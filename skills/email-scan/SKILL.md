---
name: email-scan
description: Use when the user wants to scan their inbox for anything new worth knowing, or to pull a specific email into their second brain — "scan my recent emails", "anything new in my inbox I should know about?", "pull that Delta confirmation into my second brain", "save the landlord's email about the lease". Surfaces actionable/notable mail and extracts emails into Raw.
---

# email-scan

Interactive entry point for email scanning and extraction. Two modes: **scan-now**
(classify the recent window, surface New / Reminders, auto-extract high-confidence
confirmations) and **extract-this-email** (locate an email the user references,
confirm the match, extract it into Raw). Reuses the exact pipeline `today` runs
unattended, but may prompt and do heavier work.

**The entire behavioral contract lives in `../shared-references/email.md`** — the
tool abstraction, scan window + the AGENTS *Agent state* block (`last_scan_ts`),
scope filters, the two-stage pipeline, classification, the extraction → Raw shape,
dedup, the 📧 rendering convention, and Gmail quirks/errors. This skill orchestrates
that contract interactively and **does not restate it**. Consult
`../shared-references/schema.md` for the Raw schema, the `email`/`preferences.email_tool`
config keys, and the AGENTS *Agent state* block shape;
`../shared-references/notion-conventions.md` for MCP quirks (rate limits, async
writes, Referencing pages inline); and `../shared-references/durability-modes.md` for
why `last_scan_ts` is state, not config.

## Behavior

### 1. Load config or discover

Same pattern as `today`/`capture`: read `config.json` in the launch folder if
present and valid — `inbox.data_source_url`, `inbox.triage_values`,
`preferences.timezone`, `preferences.email_tool`, the `email` block,
`tasks_personal`/`shared_spaces[].tasks` (for the relevance signal), `agents_page`.
If no `config.json`, fall back to discovery: `notion-search` for the `Second Brain`
root, `notion-fetch` its `AGENTS` page, read the fenced config block (per
`schema.md`). If neither resolves: **fail loudly**, point at `setup`, and **stop**.

Resolve the email tool per `email.md`'s **Email-tool abstraction**. If none is
available, say the connector isn't available, point at `/mcp`, and stop (this skill
is explicitly about email, so — unlike `today` — there is nothing else to do).

### 2. Decide mode from the request

- The user references a **specific email** ("the Delta confirmation", "that email
  from the landlord about the lease", a pasted Gmail link / message-id) →
  **extract-this-email** (§4).
- The user asks to **scan / catch up** ("scan my inbox", "anything new?") or the
  request is unspecific → **scan-now** (§3).

### 3. Scan-now

Run the pipeline defined in `email.md` end to end: read `last_scan_ts` from the
AGENTS *Agent state* block; compute the window (apply the **gap guard** — since this
is interactive, **ask** which window to use when the gap exceeds
`email.window_days_cap`); build the scoped query; stage-1 `search_threads`; classify;
stage-2 `get_thread` `FULL_CONTENT` for candidates; auto-extract high-confidence rows
into Raw (unless `email.auto_extract` is `false`, in which case surface them); render
the 📧 section (New / Reminders / auto-extracted / degradations) per `email.md`.

Because this is interactive, you **may** additionally:
- Offer to extract any **surfaced** item on request (turn a Reminder/New item into a
  Raw row via the extraction shape in `email.md`).
- Ask before extracting a **borderline** item rather than guessing.

Advance `last_scan_ts` to "now" (UTC instant) in the AGENTS *Agent state* block
**only after the scan succeeds** (per `email.md`).

### 4. Extract-this-email

The user references one email in natural language or pastes a Gmail link / message-id.

1. **Locate it.** Translate the reference into a Gmail query and `search_threads`
   (e.g. `from:delta subject:confirmation`, or fetch by message-id / thread-id from a
   pasted link). Use `get_thread` `FULL_CONTENT` on the best match(es) to read the
   body and attachments.
2. **Confirm the match.** Echo the sender + subject + a one-line summary and ask the
   user to confirm before writing. On **no match** or **multiple plausible matches**,
   ask rather than guessing.
3. **Extract.** On confirmation, create the Raw row per `email.md`'s **Extraction →
   Raw row** shape (`Source` = the Gmail deep link; key facts + attachment pointers in
   the body; `Triage` = the resolved `new` value). Report the created row with a link.

A single targeted extract does **not** advance `last_scan_ts` (per `email.md`) — only
a scan-now does.

### 5. Errors

Per `email.md`'s **Gmail quirks & errors**: Gmail/Notion MCP unavailable or
unauthenticated → say so, point at `/mcp`, stop; `429` → honor `Retry-After`; epoch
`after:` rejected → date-granular fallback + note; partial failure → report what
succeeded, don't advance `last_scan_ts`. Never fake a scan or a created row.

## Smoke test

Smoke test (run against a live Notion + Gmail workspace):
- **Scan-now:** seed (a) an unread actionable email — a direct ask with a deadline,
  (b) a read email you have **not** replied to that looks important, (c) a read email
  you **have** replied to (has `SENT`), (d) a Promotions email, and (e) a
  booking/receipt-style confirmation. Run `email-scan` scan-now.
  Assert: (a) appears under **New**; (b) appears under **Reminders (read, no reply)**;
  (c) is suppressed (handled); (d) never appears; (e) creates a Raw row (`Source` =
  Gmail thread link, `Triage` = the resolved `new` value, key facts + any attachment
  pointers in the body) and is noted in the 📧 section. Re-run same day → near-empty
  window, no duplicate Raw row. `last_scan_ts` in the AGENTS *Agent state* block is a
  UTC instant with offset afterward.
- **Watch/ignore:** add an `email.ignore` sender → a matching email is suppressed; add
  an `email.watch` topic → a topical-but-not-actionable match is **surfaced**, while a
  clearly-actionable one is **auto-extracted** (per `email.md`'s **Classification**).
- **Extract-this-email:** reference a specific email by description ("the Delta
  confirmation"); assert the skill locates it, confirms the match, then creates the
  Raw row — and does **not** advance `last_scan_ts`.
- **Graceful stop:** with no email tool available, `email-scan` says the connector isn't
  available and points at `/mcp` (no fake scan).
- **Timezone safety:** with a non-UTC `preferences.timezone`, the Gmail query uses
  epoch `after:` and human output renders in the preference timezone; the stored
  `last_scan_ts` is a UTC instant with offset.
