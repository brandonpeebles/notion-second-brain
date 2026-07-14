# Email scan & extract — contract

Single source of truth for how `today` and the `email` skill reach Gmail, scan a
time window, classify threads, surface them, and extract emails into Raw. Both
skills **point here** and must not restate this contract (repo shared-ref
convention). Config **shape** lives in `schema.md`; this file owns the
**behavior**. Raw property names and the `Triage` value map come from `schema.md`;
task/project relevance uses the task query `today` already runs (see below).

## Email-tool abstraction

- Resolve the email source from `preferences.email_tool`: `null` → auto-detect the
  Gmail connector (`mcp__claude_ai_Gmail__*`) in the session; `"gmail"` → pin it.
- **Graceful omit.** If no email tool is set in config **and** none is available in
  the session, omit the 📧 section / decline the scan — a complete result, never an
  error (same rule as `today`'s Calendar section). The interactive `email` skill
  says the connector isn't available and points at `/mcp`.
- **Read-only except the deliberate creation of a Raw row.** No labels, no
  mark-as-read, no archiving — no mailbox mutation of any kind. The only write email
  ever causes is a Raw row on extraction.

## Scan window (timezone-safe)

- Scan `after:<last_scan_ts>`.
- **`last_scan_ts` is an absolute UTC instant** (ISO-8601 with explicit offset, e.g.
  `2026-07-13T18:30:00Z`) — never a wall-clock local time. An instant is one unique
  moment regardless of the user's location or what the session clock reads, so
  crossing timezones (or a UTC session clock in cloud/Routine/Cowork vs. local on a
  laptop) cannot corrupt the boundary.
- **Gmail query uses the epoch-seconds form `after:<epoch>`** (converted from the
  stored instant), *not* `after:YYYY/MM/DD` — Gmail's date operators are interpreted
  in the mailbox timezone, whereas epoch seconds are absolute. **Fallback** if a
  surface rejects epoch: date form floored to the day in `preferences.timezone`
  (coarser — may re-surface part of a day; note it as a degradation).
- **Gap guard.** Compute the gap as an instant−instant duration
  (`now_utc − last_scan_ts`), never by diffing date strings. If the gap exceeds
  `email.window_days_cap` (default 3 days):
  - **Interactive:** ask which window to use before scanning.
  - **Routine / non-interactive:** cap the window at `window_days_cap` and **flag
    it** in the brief (never prompt).
- **First scan** (no `last_scan_ts` yet): treat as "no prior scan" — use the
  `window_days_cap` window; interactive callers may confirm it.
- **"now"** (for advancing the value) is captured as a UTC instant and stored
  normalized to UTC. Never reinterpret a bare local time as UTC or vice-versa.
- **Human-facing rendering only** ("scanning since <when>", brief text) converts to
  `preferences.timezone`. The stored/compared value stays UTC.

## Agent state block (last_scan_ts)

- `last_scan_ts` is **runtime state, not config** — it lives in a dedicated fenced
  ```json block under an `## Agent state` heading on the **AGENTS page**, separate
  from the config block: `{ "email": { "last_scan_ts": "<UTC instant>" } }` (shape
  in `schema.md`). Deliberately not in `config.json` (so it doesn't churn git daily)
  and not in the AGENTS config block (so it's exempt from the config-sync invariant
  — see `durability-modes.md`). Identical in `durable` and `ephemeral` modes.
- **Read:** `notion-fetch` the AGENTS page; parse the `## Agent state` fenced block.
  Absent block or absent `email.last_scan_ts` → first scan (window above).
- **Write:** advance to "now" (UTC instant) **only after a successful scan** —
  rewrite the fenced block in place with `update_content`; create it with
  `insert_content` under an `## Agent state` heading if absent (lazy creation). A
  crashed/partial run leaves `last_scan_ts` untouched so the window is retried.
- **Single targeted extract** (the `email` skill's extract-this-email mode) does
  **not** advance `last_scan_ts` — only a full scan does.

## Scope filters (defaults; all overridable)

Base query (default): `in:inbox -category:promotions -category:social`, **read +
unread both included.** Composable toggles from config:

- `email.unread_only: true` → append `is:unread`.
- `email.important_only: true` → append `is:important` (Gmail Priority-Inbox
  "important" conversations).
- Toggles compose (both → `is:important is:unread`).
- `email.scan_query` (non-null) → **replaces the base query wholesale** (toggles and
  default category exclusions no longer apply — the user takes full control).
  `after:<window>` is still appended.

## Two-stage pipeline (efficiency)

1. **Cheap pass:** one/few paginated `search_threads` calls over the window →
   classify each thread from subject/snippet/sender/labels + intrinsic patterns +
   active-tasks/projects match + watch/ignore lists. **No bodies fetched.**
2. **Rich pass (candidates only):** for auto-extract candidates and
   read-and-plausibly-important candidates, `get_thread` `FULL_CONTENT` to pull key
   facts, attachment names, and reply-state (`SENT` label). Ignored mail is never
   fetched beyond stage 1.

## Classification (default; tunable both directions)

Three outcomes per thread.

**Ignore** (never surfaced, never fetched beyond stage 1):
- Promotions / Social (excluded at query level by default).
- `email.ignore` matches — senders or topics (recruiters / cold outreach, etc.).
- Obvious bulk / newsletter / automated no-reply blasts.

**Auto-extract** (high bar — writes a Raw row unattended):
- Structured **transactional confirmations**: booking / reservation / itinerary,
  order / receipt / invoice / payment, appointment confirmations — detected by
  intrinsic patterns (sender, subject keywords, presence of confirmation numbers /
  dates / amounts).
- A clearly-actionable message on a thread matching an **active Project / open Task**
  (from the task query `today` already ran) or an `email.watch` entry.
- Read-state does **not** change the auto-extract bar (timestamp dedup makes re-runs
  safe).
- Controlled by `email.auto_extract` (default `true`). When `false`, these are
  **surfaced** instead of written.

**Surface** (listed in the brief, no write):
- **Unread + actionable/relevant** → **New**.
- **Read + you already replied** (a thread message carries the `SENT` label) →
  **handled**; suppressed from the surface list. Still eligible for auto-extract if
  confirmation-grade.
- **Read + no reply sent + looks important/actionable/extract-worthy** →
  **Reminders (read, no reply)**.
- "Actionable/relevant" = a direct ask/question to the user, a stated deadline, or
  topical overlap with active tasks/projects/watch-list — but not confirmation-grade.

**Relevance signal:**
- **Free:** match against the task/project titles the caller **already** queried this
  run (no extra queries).
- **Opt-in:** `email.wiki_match: true` adds a per-candidate `notion-search` across
  Wiki/Raw. Off by default (costly / noisy).

**Reply-state detection:** the **`SENT` label on any message in the thread** —
identity-agnostic, no hardcoding the user's address. Checked only in stage 2, for
read-and-plausibly-important candidates (adds no cost to ignored/unread-only flows).

## Extraction → Raw row

Write with `notion-create-pages` into `inbox.data_source_url`, using the canonical
Raw property names (`Name`/`Type`/`Source`/`Triage`/`Captured`) per `schema.md`, and
the `Triage` **value** resolved from `inbox.triage_values` (default `New`):

- `Name` — cleaned subject.
- `Type` — best guess among `Task` / `Note` / `Idea` / `Reference` (confirmations →
  `Reference`).
- `Source` — the **Gmail deep link** to the thread.
- `Triage` — the resolved `new` value.
- `Captured` — automatic (`created_time`); never set manually.
- **Body** — a short summary + extracted **key facts** (dates, amounts, confirmation
  numbers, sender), and, when the email has attachments, a **pointer list**: each
  attachment's filename/type + the thread link. **No file embedding** — the Gmail
  connector has no attachment-download tool, so attachments are pointers, not embeds.
  (If the connector later exposes attachment bytes, embedding via
  `notion-create-attachment` is a drop-in upgrade.)

Extracted rows then flow through the existing `triage` sweep like any capture — no
special downstream handling.

## Dedup (timestamp-only)

The advancing `last_scan_ts` means a re-run scans a near-empty window, so duplicate
rows aren't created. **No Gmail writes, no Raw lookups.** (Trade-off: a partial run
that failed before advancing `last_scan_ts` re-scans the same window next time —
the safe direction; the high auto-extract bar plus near-empty windows keep any
duplication negligible.)

## Rendering the 📧 section

The **📧 From your inbox** section renders two groups — **New** and **Reminders
(read, no reply)** — plus a note of any auto-extracted rows (with links) and any
degradations (window capped, epoch fallback, email tool absent, window truncated).

**Per-email bullet** (mirrors `today`'s per-task bullet convention in
`notion-conventions.md`, but a Gmail thread is an external URL, not a Notion page —
so use a plain markdown link in **both** chat and the Journal body, never a
`<mention-page>`): `- [<cleaned subject>](<gmail thread link>) — <sender> · <one-line
why it surfaced>`. Example:
`- [Re: lease renewal terms](https://mail.google.com/…) — landlord · asks for a decision by Fri`.

Auto-extracted rows are noted under the section as
`- Extracted to Raw: [<subject>](<gmail link>)` and, per the Referencing-pages-inline
convention, may additionally link the created Raw row (`<mention-page>` in the
Journal body, `[title](url)` in chat).

## Gmail quirks & errors

- **`search_threads`:** `pageSize` ≤ 50, paginate via `pageToken`; full Gmail query
  grammar (`category:`, `is:read`/`is:unread`, `is:important`, `has:attachment`,
  `from:`/`to:`, `OR`/grouping). Returns subject + snippet + sender + recipients per
  message — **no bodies**.
- **`get_thread`/`get_message` `FULL_CONTENT`:** bodies + attachment IDs/filenames +
  per-message labels (incl. `SENT`).
- **No attachment-download tool** exists in the connector → attachments are pointers,
  never embeds (above).
- **Pagination cap:** paginate up to **5 pages (250 threads)** per scan. If the cap
  is hit, **report** that the window was truncated (log/note) rather than silently
  dropping mail.
- **No email tool available:** omit / decline gracefully; never error.
- **Gmail MCP unavailable / unauthenticated:** say so plainly, point at `/mcp`, and
  **stop** — never fake a scan or report emails that weren't fetched (same rule as
  `today`'s Notion errors).
- **Rate limiting (429):** honor `Retry-After` / back off (per
  `notion-conventions.md`, applied to Gmail too).
- **Epoch `after:` rejected by the surface:** fall back to date-granular
  `after:YYYY/MM/DD` in `preferences.timezone` and note the degradation.
- **Partial scan failure:** advance `last_scan_ts` only on success; report what
  succeeded, flag what failed.
