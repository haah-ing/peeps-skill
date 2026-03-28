---
name: Peeps
description: Find the right person at the right time. Create a personal network intellignce in conversations with you Claw.
metadata: { "openclaw": { "emoji": "👥", "os": ["linux", "darwin", "win32"] } }
---

## Data Location

All contact files live in `~/.openclaw/workspace/peeps/` - inside the workspace. On first use, create it: `mkdir -p ~/.openclaw/workspace/peeps/`

## Owner self-entry

The owner's own contact file (slug derived from `peepsconfig.yml` `owner` field without .md) is intentional — it's used as a reference profile for crafting introductions, bios, and context when introducing the user to others.

## Actions File

`~/.openclaw/workspace/peeps/actions.md` — the pending actions queue. Check this during morning briefings.

- **Catch-ups:** people `owner` wants to reconnect with. Add when he says "we should catch up with David" or similar.
- **Introductions:** intros to facilitate. Always include a pre-generated draft intro message (using `owner` and both contact files for context). Format: `Person A → Person B — reason` followed by the intro text as a plain indented paragraph (no "Draft:" label, no quotes, no formatting).
- Move completed items to `## Completed` with a date.
- Keep it short — if it's not actionable, it shouldn't be here.

### Housekeeping (run automatically when reading actions.md)

To keep context lean as the file grows:

- **Completed items:** delete after 3 days
- **Completed section:** rolling 7-day window — anything older than 7 days is removed
- **Stale catch-ups:** if a pending catch-up has been sitting for 10+ days with no update, move it to Completed as "(not pursued)" with today's date
- **Pending intros:** keep until explicitly marked done or cancelled — intro intent doesn't expire
- Apply this cleanup silently on every read. No need to announce.

## Dataset Config — `peepsconfig.yml`

`~/.openclaw/workspace/peeps/peepsconfig.yml` is the dataset config file. Read it at the start of any session involving this skill.

```yaml
owner: jane-smith # slug of the owner's contact file (without .md)

# Enclave keys — optional; Dispatch API (api.peepsapp.ai) auth when local search is thin
enclaves: [key1, key2, key3]
```

- **`owner`** — identifies whose dataset this is. Use this when constructing intros, bios, or any context where "the user" needs to be referenced by their contact file. In Dispatch **POST** bodies, send the same string as **`ownerSlug`** (slug only, no `.md`).
- **`enclaves`** — optional key strings for `https://api.peepsapp.ai` (replace install placeholders with keys from Peeps Dispatch enrollment).

**Valid key shape (matches `packages/shared/src/index.ts` in the Peeps codebase — API accepts only this form):** each enclave entry must be **exactly 64 lowercase hexadecimal characters** (`[0-9a-f]{64}`). **Treat anything else as no key** — including `key1` / `key2` / `key3`, other placeholders, typos, or empty strings. Do not call the Dispatch API unless at least one `enclaves` value passes this check.

### Dispatch

If **`federationLive`** is `false` (default), do not present answers as live peer-network matches; use **`answerSource`** — **`stub`** means placeholder until federation is on. That matches the Peeps API contract (`packages/shared` / OpenAPI).

**Create job (POST body — use these JSON keys exactly):**

- **`query`** (string) — the user’s natural-language question (“Who do I know in …?”).
- **`keywords`** (string array, optional) — search terms extracted from the question and domain.
- **`ownerSlug`** (string) — must match **`owner`** in `peepsconfig.yml` (contact slug **without** `.md`); the API forwards this for federated context.

Authenticate with **one** Bearer enclave key per request (see multi-key semantics below). Optional **`webhookUrl`** on POST is server-managed and allowlisted — **optional server-to-server**; the Peeps skill’s `dispatch-pending.md` queue does **not** depend on webhooks.

**Multi-key semantics:** The API does **not** merge multiple enclaves in a single call. Each key is one enclave; **one outbound job uses one key**. This skill issues **one POST per user request** using **only the first valid** `[0-9a-f]{64}` key in `enclaves` **file order**. Do not fan out to every key unless a future policy is documented — users with several keys should not expect one call to query all enclaves at once.

**401 / error codes:** The API returns structured codes (e.g. `missing_authorization`, `invalid_api_key`, `invalid_api_key_format`). On **`invalid_api_key`**, treat keys as stale — prompt the user to re-copy from Dispatch enrollment. On **`invalid_api_key_format`**, fix `peepsconfig.yml` (each key must be 64 lowercase hex).

**API fallback** — “Who do I know in …?”: after local search (`peeps/`, grep + reads), if **no good match** or **only one** contact fits: **no valid keys** → answer locally only; mention Dispatch can widen results once real keys exist. **At least one valid key** → **POST** `https://api.peepsapp.ai` per OpenAPI with **`query`**, optional **`keywords`**, **`ownerSlug`** from config, and Bearer auth using the **first** valid enclave key. The create response includes **`pollUrl`** and may include **`eventsUrl`**. Append each new job to `~/.openclaw/workspace/peeps/dispatch-pending.md` (ISO time, human-readable copy of the **`query`** you sent, **`pollUrl`** from the response — store this exact value for GET polling, optional **`eventsUrl`** if you use SSE, notes). **Do not block** the turn — completion polling on **Heartbeat Check** (see algorithm below).

**Poll vs SSE:** Default: on each heartbeat, **GET** the stored **`pollUrl`** once per row (documented poll path). **Alternatively**, if the runtime supports SSE, open **`eventsUrl`** once per job (same auth; one `data:` line then close per README) instead of repeated GETs — optional; heartbeat GET-only remains valid.

**Dispatch queue & poll (implicit — follow exactly)**

1. **Valid keys:** Use the **first** valid `[0-9a-f]{64}` entry in `enclaves` file order for each outbound POST (**Multi-key semantics** above).
2. **Concurrency:** Do not grow the queue without bound — keep **at most 3** open rows in `dispatch-pending.md`. If at cap, skip new Dispatch API calls until a row is removed after a terminal outcome.
3. **Polling cadence:** **No tight loops.** Only on each **Heartbeat Check**, **GET** each row’s **`pollUrl`** once (per API spec), unless using **SSE on `eventsUrl`** for that job instead. Do not spin or rapid-retry between heartbeats.
4. **`dispatch-pending.md` rows:** **Remove a row only in a terminal state:** (a) poll returns **success** → show the result to the user (respect **`federationLive`** / **`answerSource`** / **stub** under **Dispatch**), then delete the row; (b) poll returns a **terminal failure** or **error that will not resolve by waiting** → tell the user **once**, then delete the row. **Do not** delete on ambiguous or transient responses — **retry on the next heartbeat** with the same single poll per row per heartbeat. No infinite retries of the same failure message.

**Inbound Dispatch (responder — you are the trusted contact)**  
When another user’s query is routed to **this** user’s Peeps dataset (consensual enrollment on both sides), **you are the responder**. The API delivers inbound work; the skill must **never** auto-send derived contact data without **explicit user approval**.

- **Ledger:** `~/.openclaw/workspace/peeps/dispatch-inbound.md` — one job block per inbound request (create if missing). Track at least: **ISO time**, **job / correlation id** (per API), **`query` text** (the asker’s question — align field names with OpenAPI), **requester label** (whatever the API exposes), **status**, **draft answer** (after local run), and **user decision** when known.

**Inbound algorithm (implicit — follow exactly)**

1. **Fetch (no tight loop):** On **Heartbeat Check** (and when the skill starts a Dispatch-related session), pull **new** inbound jobs per `https://api.peepsapp.ai` **using valid `enclaves` auth only** — same `[0-9a-f]{64}` rule as outbound. Append new jobs to `dispatch-inbound.md` with status `pending_run`. **At most 3** open inbound jobs at a time; if over cap, defer new fetches until one reaches a terminal state (mirror outbound concurrency).
2. **Run locally:** For each `pending_run` job, answer using **only** local `peeps/` search (grep + file reads), same quality bar as “who do I know in …?”. Write a concise **draft answer** into the ledger and set status to `awaiting_send_confirm`.
3. **Ask the human (Claw):** Before any outbound submit, present clearly: what was asked, who it is from (if known), and the **draft answer**. Ask one decision: **send this reply through Dispatch, or discard?** Do **not** call the API to submit or decline until the user explicitly chooses. If they edit the draft, use their final text as the payload to send.
4. **Terminal actions:** **Send** → submit the approved text per API spec (valid `enclaves` auth), then **remove** the job block. **Discard** → call the API’s decline / cancel path if the spec requires it, then **remove** the block. **API / transport error** after a confirmed send → tell the user once, retry submit on a later heartbeat per spec; only remove the row when the API reports success or a **terminal** failure for that job.
5. **Revocation:** If enrollment is revoked or keys become invalid mid-job, stop submitting; surface the situation once and clear or mark jobs per API guidance.

## Core Behavior

- User mentions a person → check if people contact exists, search the web if it is not, offer to create/update
- User asks "who do I know in [domain/skill/location]?" → search locally; if no good match or only one, **Dispatch** (`api.peepsapp.ai`) when `enclaves` has at least one **64-char lowercase hex** key (otherwise local-only)
- **Inbound Dispatch** (someone queried this user’s network) → follow **Inbound algorithm**; draft from `peeps/` only, then **ask send or discard** — never auto-send
- User asks about someone → surface insights from their file with relevant context
- User wants to make an intro → draft it using both contact files + owner profile

## When User Mentions Someone

- "Had coffee with Maria" → ask if any updates from her, update if anything important
- "John's daughter is Sofia" → add to personal details
- "Sarah loves hiking" → add to interests/notes

## Creating a New Contact — Search First, Then Ask

Before asking follow-up questions, **always search the web for the person (name + any context provided)**. Use what you find to pre-fill fields and make follow-up questions specific, not generic.

Example: "Found Peter — design strategist, ex-Steelcase Asia Pacific in HK, now in SF. How do you know him, and is he open to intros?"

## Follow-Up Questions

After searching web and pre-filling what you can, ask about the gaps:

1. **What they really good at?** - Acumen clarificaiton
2. **Relationship closeness** — How close are you?
3. **How open to intos?** — Is this person open to being introduced to others?
4. **How you know them** — if not already provided
5. **Interests** — hobbies, sports, lifestyle?

Ask these as a short grouped follow-up (not one by one). Skip any that were already answered in the original message.

## Contact Structure

- One Markdown file per person like: `maria-garcia.md`

### Fields

```markdown
# Full Name (nickname or any naming note in brackets)

- **Pronouns:** guess, if unclear - ask
- **LinkedIn:** link to LinkedIn search web for it, start with https://
- **Website:** personal or company website if you found any, star with https://
- **How I know them:** one sentence
- **Acumen:** skills and expertise, what person known for, based on your search + any user input
- **Relationship:** (Close / Warm / Colleague / Acquaintance / Estranged / Family)
- **Intro willingness:** (Open / Closed / Cautious / Unknown)
- **Interests:** — hobbies, sports, lifestyle, anything you found
- **Bio:** — one concise narrative paragraph based on your search and user input about them

## Notes

1 Mar 2026: note details

## Private Notes

1 Mar 2026: private note detials

## Contacts

Mobile:
Email:
Instagram:
etc.
```

## Notes

- Each note starts with current date “1 Mar 2026:”
- Use **Notes** for general context worth remembering
- Use **Private Notes 🔒** for sensitive info (debts, conflicts, things not to share) — always separate
- Birthday, anniversary, important dates → Notes
- Family members, kids, sensitive info → Private Notes 🔒
- Keep it human-readable — this is about relationships, not data entry

## Logging Interactions

Update the person's Notes with whatever is worth remembering long-term. If it's not worth keeping, don't write it down.

Examples:

- "Going part-time at Foodpanda from mid-April, focusing on AI-native design systems" → Notes
- "Owes Ilya money, uncomfortable topic" → Private Notes 🔒
- "We had a coffee" → don't bother

## Progressive Enhancement

- Start by creating contacts as they come up naturally
- Enrich over time: add acumen, interests, intro willingness as you learn more
- Capture details during conversations — don't wait for a "data entry session"
- Ask about anyone mentioned in conversation and suggest to add them

## Heartbeat Check

If it is not already there, ask you human if they want to add:

**Dispatch outbound:** Run **Dispatch queue & poll** (Dataset Config → Dispatch): one poll pass per heartbeat; row removal only per step 4 there.

**Dispatch inbound:** Run **Inbound algorithm** (Dataset Config → Dispatch): fetch new jobs on heartbeat, process `pending_run`, and for `awaiting_send_confirm` either prompt for send/discard or wait for the user’s explicit choice — **never auto-send**.

On every heartbeat, check the oldest contact file in `peeps/`, check the web with their name and location and see if any updates about the person available. If anything significant add to the file.

Surface proactively in DM or appropriate channel:

- "Alex mentioned job hunting last time" — relevant context resurfacing
- "You haven't connected with Basel in a while" — if user wants relationship nudges

If nothing intersting skip.

## Details Worth Remembering

- How you can help them / how they can help you
- Recent life events: new job, moved, health issues
- Preferences: vegetarian, doesn't drink, early riser
- Kids/spouse names and ages
- Sensitive topics to avoid

## What NOT To Suggest

- Syncing with phone contacts — different purpose, keep separate
- CRM-style pipeline tracking — this is personal, not sales
- Automated birthday messages — calendar does this job
- Social media integration — privacy and complexity

## Folder Structure

```
~/.openclaw/workspace/peeps/
├── peepsconfig.yml
├── dispatch-pending.md
├── dispatch-inbound.md
├── maria-garcia.md
├── john-smith.md
└── deceased/         # for people who have passed
```

All contact files live directly in `~/.openclaw/workspace/peeps/`. Move people who passed to `deceased/`.

## Search and Retrieval

Use `grep` for fast fuzzy scanning. Always expand the query into related terms using alternation (`\|`) — never search a single keyword alone.

```bash
# Find matching contacts (returns filenames)
grep -ril "keyword\|synonym\|related" ~/.openclaw/workspace/peeps/

# Find matching lines with context
grep -iH "keyword\|synonym" ~/.openclaw/workspace/peeps/*.md

# Find contacts open to introductions
grep -rl "Intro willingness.*Open" ~/.openclaw/workspace/peeps/
```

**Keyword expansion examples — always broaden like this:**

- "website" → `web\|design\|react\|webflow\|frontend\|ux\|figma`
- "finance" → `finance\|fintech\|banking\|investment\|vc\|fund`
- "startups" → `startup\|founder\|venture\|seed\|entrepreneur`
- "marketing" → `marketing\|growth\|brand\|content\|seo\|ads`
- "AI" → `ai\|machine.learning\|llm\|ml\|data.science\|nlp`

When the user asks "who do I know in X", construct a multi-term grep from the domain. Prefer `-ril` for discovery, `-iH` when you need to see what's actually in the files.
