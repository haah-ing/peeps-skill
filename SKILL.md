---
name: Peeps
description: Find the right person at the right time. Create personal network intelligence in conversations with your Claw.
metadata: { "openclaw": { "emoji": "👥", "os": ["linux", "darwin", "win32"] } }
---

## Peeps — local workspace & contacts

### Data Location

All contact files live in `~/.openclaw/workspace/peeps/` — inside the workspace. On first use, create it: `mkdir -p ~/.openclaw/workspace/peeps/`

### Owner self-entry

The owner's own contact file (slug derived from `peepsconfig.yml` `owner` field without `.md`) is intentional — it's used as a reference profile for crafting introductions, bios, and context when introducing the user to others.

### Actions File

`~/.openclaw/workspace/peeps/actions.md` — the pending actions queue. Check this during morning briefings.

- **Catch-ups:** people `owner` wants to reconnect with. Add when he says "we should catch up with David" or similar.
- **Introductions:** intros to facilitate. Always include a pre-generated draft intro message (using `owner` and both contact files for context). Format: `Person A → Person B — reason` followed by the intro text as a plain indented paragraph (no "Draft:" label, no quotes, no formatting).
- Move completed items to `## Completed` with a date.
- Keep it short — if it's not actionable, it shouldn't be here.

#### Housekeeping (run automatically when reading actions.md)

To keep context lean as the file grows:

- **Completed items:** delete after 3 days
- **Completed section:** rolling 7-day window — anything older than 7 days is removed
- **Stale catch-ups:** if a pending catch-up has been sitting for 10+ days with no update, move it to Completed as "(not pursued)" with today's date
- **Pending intros:** keep until explicitly marked done or cancelled — intro intent doesn't expire
- Apply this cleanup silently on every read. No need to announce.

### Dataset Config — `peepsconfig.yml`

`~/.openclaw/workspace/peeps/peepsconfig.yml` is the dataset config file. Read it at the start of any session involving this skill.

```yaml
owner: jane-smith # slug of the owner's contact file (without .md)

# Enclave keys — optional; used only for Dispatch API (see Appendix)
enclaves: [key1, key2, key3]
```

- **`owner`** — identifies whose dataset this is. Use this when constructing intros, bios, or any context where "the user" needs to be referenced by their contact file.
- **`enclaves`** — optional. Dispatch HTTP calls require **64-character lowercase hex** keys; placeholders like `key1` are not valid. Full outbound/inbound behavior is in **Appendix: Dispatch**.

### Core Behavior

- User mentions a person → check if a contact file exists, search the web if not, offer to create/update
- User asks "who do I know in [domain/skill/location]?" → search locally first; **Dispatch** outbound only when keys and conditions in the **Appendix** are met
- **Inbound Dispatch** (someone queried this user's network) → **Appendix: Dispatch (Inbound)**; draft from `peeps/` only, then ask send or discard — never auto-send
- User asks about someone → surface insights from their file with relevant context
- User wants to make an intro → draft it using both contact files + owner profile

### When User Mentions Someone

- "Had coffee with Maria" → ask if any updates from her, update if anything important
- "John's daughter is Sofia" → add to personal details
- "Sarah loves hiking" → add to interests/notes

### Creating a New Contact — Search First, Then Ask

Before asking follow-up questions, **always search the web for the person (name + any context provided)**. Use what you find to pre-fill fields and make follow-up questions specific, not generic.

Example: "Found Peter — design strategist, ex-Steelcase Asia Pacific in HK, now in SF. How do you know him, and is he open to intros?"

### Follow-Up Questions

After searching the web and pre-filling what you can, ask about the gaps:

1. **What are they really good at?** — Acumen clarification
2. **Relationship closeness** — How close are you?
3. **How open to intros?** — Is this person open to being introduced to others?
4. **How you know them** — if not already provided
5. **Interests** — hobbies, sports, lifestyle?

Ask these as a short grouped follow-up (not one by one). Skip any that were already answered in the original message.

### Contact Structure

- One Markdown file per person like: `maria-garcia.md`

#### Fields

```markdown
# Full Name (nickname or any naming note in brackets)

- **Pronouns:** guess, if unclear - ask
- **LinkedIn:** link to LinkedIn search web for it, start with https://
- **Website:** personal or company website if you found any, start with https://
- **How I know them:** one sentence
- **Acumen:** skills and expertise, what person known for, based on your search + any user input
- **Relationship:** (Close / Warm / Colleague / Acquaintance / Estranged / Family)
- **Intro willingness:** (Open / Closed / Cautious / Unknown)
- **Interests:** — hobbies, sports, lifestyle, anything you found
- **Bio:** — one concise narrative paragraph based on your search and user input about them

## Notes

1 Mar 2026: note details

## Private Notes

1 Mar 2026: private note details

## Contacts

Mobile:
Email:
Instagram:
etc.
```

### Notes (guidance)

- Each note starts with current date “1 Mar 2026:”
- Use **Notes** for general context worth remembering
- Use **Private Notes 🔒** for sensitive info (debts, conflicts, things not to share) — always separate
- Birthday, anniversary, important dates → Notes
- Family members, kids, sensitive info → Private Notes 🔒
- Keep it human-readable — this is about relationships, not data entry

### Logging Interactions

Update the person's Notes with whatever is worth remembering long-term. If it's not worth keeping, don't write it down.

Examples:

- "Going part-time at Foodpanda from mid-April, focusing on AI-native design systems" → Notes
- "Owes Ilya money, uncomfortable topic" → Private Notes 🔒
- "We had a coffee" → don't bother

### Progressive Enhancement

- Start by creating contacts as they come up naturally
- Enrich over time: add acumen, interests, intro willingness as you learn more
- Capture details during conversations — don't wait for a "data entry session"
- Ask about anyone mentioned in conversation and suggest adding them

### Heartbeat Check

If it is not already there, ask your human if they want to add:

**Dispatch (outbound + inbound):** Follow **Appendix: Dispatch** — one poll/fetch pass per heartbeat; row removal only as specified there.

On every heartbeat, check the oldest contact file in `peeps/`, check the web with their name and location and see if any updates about the person are available. If anything significant, add to the file.

Surface proactively in DM or appropriate channel:

- "Alex mentioned job hunting last time" — relevant context resurfacing
- "You haven't connected with Basel in a while" — if user wants relationship nudges

If nothing interesting, skip.

### Details Worth Remembering

- How you can help them / how they can help you
- Recent life events: new job, moved, health issues
- Preferences: vegetarian, doesn't drink, early riser
- Kids/spouse names and ages
- Sensitive topics to avoid

### What NOT To Suggest

- Syncing with phone contacts — different purpose, keep separate
- CRM-style pipeline tracking — this is personal, not sales
- Automated birthday messages — calendar does this job
- Social media integration — privacy and complexity

### Folder Structure

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

### Search and Retrieval

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

---

## Appendix: Dispatch

Opt-in at dispatch.peepsapp.ai; Peeps works fully without it. Prefer **`GET https://api.peepsapp.ai/openapi.json`** if field names drift.

### Client policy

**Pending row cap:** keep at most **3** open rows in **`dispatch-pending.md`** and at most **3** in **`dispatch-inbound.md`** — client-side only, not an API requirement. If at cap, defer new work until a row clears.

### Federation & answer presentation

- **Stub rule (single rule for copy):** if **`answerSource === "stub"`** or **`stub === true`**, do **not** present the answer as live federated peer-network data — it is placeholder until real federation delivers matches.
- **`federationLive`:** use only for **deployment / product messaging** (e.g. explain when federation is not fully live). Do not restate the stub rule; rely on **`answerSource` / `stub`** for how strongly to sell an answer.

### Outbound Dispatch

**Prerequisites:** at least one **`enclaves`** value must match **`[0-9a-f]{64}`**; treat anything else as no key. In Dispatch **POST** bodies, **`ownerSlug`** must match **`owner`** in `peepsconfig.yml` (slug only, no `.md`).

**Create job (POST body — JSON keys must match API):**

- **`query`** (string) — the user’s question (never use a field named `question`).
- **`keywords`** (string array, optional).
- **`ownerSlug`** (string) — same as **`owner`** in config.

Authenticate with **one** Bearer enclave key per request. **`webhookUrl`** is optional server-to-server; this skill’s file queues do not require webhooks.

**Multi-key semantics:** one job uses one enclave key; the API does not merge keys in one call. This skill uses the **first** valid key in `enclaves` file order per outbound POST unless a future policy says otherwise.

**401 / errors:** on **`invalid_api_key`**, treat keys as stale — prompt re-copy from Dispatch enrollment. On **`invalid_api_key_format`**, fix `peepsconfig.yml`.

**When to call:** after local search (`peeps/`, grep + reads), if **no good match** or **only one** contact fits and a valid key exists → **POST** `https://api.peepsapp.ai` per OpenAPI with **`query`**, optional **`keywords`**, **`ownerSlug`**, Bearer auth. Response includes **`pollUrl`** and may include **`eventsUrl`**. Append jobs to **`dispatch-pending.md`** (ISO time, copy of **`query`**, store **`pollUrl`** for GET polling, optional **`eventsUrl`** for SSE, notes). Do not block the user’s turn — poll on **Heartbeat**.

**Poll vs SSE:** default: each heartbeat, **GET** each stored **`pollUrl`** once. Alternatively, **SSE** on **`eventsUrl`** if the runtime supports it (same auth; README-style `data:` line). Heartbeat GET-only is always valid.

**Outbound queue & poll**

1. Use the **first** valid enclave key per **Multi-key semantics**.
2. Respect **Pending row cap** for `dispatch-pending.md`.
3. **No tight loops** — only on each heartbeat, one **GET** per **`pollUrl`** (or SSE for that job), unless between-heartbeat retry after a confirmed failure path.
4. **Remove a row only in a terminal state:** success → show result (apply **Stub rule**), then delete; terminal poll failure → tell the user once, then delete. Retry ambiguous/transient responses on the next heartbeat.

### Inbound Dispatch (responder)

You are the **trusted contact** for another user’s query. **Never** auto-send derived contact data; require explicit approval before **`respond`**.

**HTTP (replace host if self-hosted; `Bearer` = valid hex key):**

| Action | Method | Path |
|--------|--------|------|
| List pending | `GET` | `https://api.peepsapp.ai/v1/dispatch/inbound/jobs?status=pending` |
| Respond | `POST` | `https://api.peepsapp.ai/v1/dispatch/inbound/jobs/{jobId}/respond` — body `{"answer":"..."}` |
| Enqueue (upstream / tests) | `POST` | `https://api.peepsapp.ai/v1/dispatch/inbound/jobs` — e.g. `{"query":"…","requesterLabel":"…"}` |

**Quick curl (debug):** replace `KEY` / host as needed.

```bash
curl -sS -X POST https://api.peepsapp.ai/v1/dispatch/inbound/jobs \
  -H "Authorization: Bearer KEY" -H "Content-Type: application/json" \
  -d '{"query":"Who do you know in fintech?","requesterLabel":"Alex"}'

curl -sS "https://api.peepsapp.ai/v1/dispatch/inbound/jobs?status=pending" \
  -H "Authorization: Bearer KEY"

curl -sS -X POST "https://api.peepsapp.ai/v1/dispatch/inbound/jobs/JOB_UUID/respond" \
  -H "Authorization: Bearer KEY" -H "Content-Type: application/json" \
  -d '{"answer":"From my notes: …"}'
```

**Ledger:** `dispatch-inbound.md` — **ISO time**, **`jobId`**, **`query`**, **`requesterLabel`**, **status**, **draft answer**, **user decision**.

**Inbound algorithm**

1. **Fetch:** on heartbeat / Dispatch session, **`GET`** `…/inbound/jobs?status=pending` with Bearer (first valid key). Merge into the ledger; **`pending_run`** for new local work. Respect **Pending row cap**.
2. **Run locally:** **`pending_run`** → answer from **`peeps/`** only → **`awaiting_send_confirm`** with a draft.
3. **Ask the human:** show **`query`**, **`requesterLabel`**, draft; **send or discard?** No **`POST …/respond`** until **send** is chosen.
4. **Terminal:** **send** → **`POST …/respond`** with `{"answer":"..."}` → remove row on success. **Discard** → remove ledger row (decline route only if OpenAPI defines one). Retry transport errors on later heartbeats until terminal outcome.
5. **Revocation:** invalid keys or revoked enrollment — stop; surface once; clear per API guidance.
