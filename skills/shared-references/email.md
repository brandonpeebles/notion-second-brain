# Email scan & extract — contract

Single source of truth for how `today` and the `email-scan` skill reach Gmail, scan a
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
  error (same rule as `today`'s Calendar section). The interactive `email-scan` skill
  says the connector isn't available and points at `/mcp`.
- **Read-only except the deliberate creation of a Raw row.** No labels, no
  mark-as-read, no archiving — no mailbox mutation of any kind. The only write email
  ever causes is a Raw row on extraction.

## Scan window (timezone-safe)

- Scan `after:<window_start>` (the Window-start rule below; `last_scan_ts` is its
  incremental floor).
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
- **Window start (day-scoped floor).** The scan's lower bound is a single derived
  instant, computed **identically** by `today` §3a and `email-scan` §3:
  ```
  capped_start = max(last_scan_ts, now_utc − window_days_cap)   # the gap guard above, unchanged
  day_start    = start of the resolved date in preferences.timezone, as a UTC instant
  window_start = min(capped_start, day_start)
  ```
  Because `window_start ≤ day_start` always, the window can **never start later than
  today's local midnight** — a re-run later in the day still re-scans the **whole day**,
  not a near-empty slice. Convert `window_start` to the Gmail `after:` form exactly as
  before (epoch seconds, with the date-granular fallback on rejection). **First scan**
  (no `last_scan_ts`): a no-op — the `window_days_cap` window already contains
  `day_start`. The gap guard, capped-window flag, epoch conversion, and UTC-instant
  storage/compare rules are all unchanged; this rule only *widens* the lower bound (down
  to today's midnight), never past it.
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
- **Single targeted extract** (the `email-scan` skill's extract-this-email mode) does
  **not** advance `last_scan_ts` — only a full scan does.
- **Role (narrowed — state it outright).** `last_scan_ts` keeps its exact definition
  above (UTC instant, advanced only on a successful full scan). Its **role** is now
  exactly two jobs: (1) the **catch-up floor** — via `capped_start` in the window rule —
  and (2) the **`· new` boundary** in the chat render (see "Rendering the 📧 section").
  It **no longer bounds what the Journal 📧 section contains** — `window_start` (a
  superset of both "today" and "since `last_scan_ts`") does. A thread older than
  `last_scan_ts` but from earlier today still belongs in today's section; it is simply
  rendered **unmarked**.

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
2. **Rich pass (candidates only):** for auto-extract candidates,
   read-and-plausibly-important candidates, and awaiting-reply candidates (tail-check,
   see "Awaiting-reply sweep"), `get_thread` `FULL_CONTENT` to pull key facts,
   attachment names, and reply-state (`SENT` label). Ignored mail is never fetched
   beyond stage 1.

## Classification (default; tunable both directions)

Three outcomes per thread.

**Ignore** (never surfaced, never fetched beyond stage 1):
- Promotions / Social (excluded at query level by default).
- `email.ignore` matches — senders or topics (recruiters / cold outreach, etc.).
- Obvious bulk / newsletter / automated no-reply blasts — **unless the thread matches an
  active Project / open Task / `email.watch`**, in which case *relevance wins*: promote
  it to **Surface → Updates** (below). The override requires a match to an **active**
  project/task/watch entry, not loose keyword overlap — so a spam blast to undisclosed
  recipients with no active-topic tie stays ignored.

**Auto-extract** (high bar — writes a Raw row unattended; **off by default**):
- **Travel / booking / itinerary confirmations** — flights, hotels, rentals, event
  tickets. These always qualify (detected by sender + subject keywords + presence of
  confirmation numbers / dates).
- **Orders / receipts / invoices / payments** qualify **only when the thread matches an
  active Project / open Task** (from the task query `today` already ran) **or an
  `email.watch` entry** — a topic-tie. **No dollar threshold**: amount alone is the wrong
  signal (payroll *income* is the largest line and the least worth capturing).
- **Routine receipts with no topic match** — coffee, rideshare, food delivery,
  app / subscription renewals, P2P transfers, payroll / income — **do not qualify**. They
  fall to **Ignore**, or **Surface** only if relevant.
- A clearly-actionable message on a thread matching an **active Project / open Task** or
  an `email.watch` entry.
- Read-state does **not** change the auto-extract bar (timestamp dedup makes re-runs
  safe).
- Controlled by `email.auto_extract` (**default `false`**). When `false` (the default),
  qualifying items are **surfaced** for on-demand extraction — the explicit `email-scan`
  extract-this-email path still writes; set it `true` to write them unattended.

**Surface** (listed in the brief, no write) — four groups, each omitted when empty:
- **New** — unread + actionable/relevant.
- **Reminders — you haven't replied** — read, *they* spoke last, no reply from you,
  looks important/actionable/extract-worthy.
- **Updates / heads-up** — relevant-but-**not**-actionable items promoted by the
  relevance override: notifications tied to an active Project / open Task / `email.watch`
  (RSVPs, invites, a CI failure on your own repo, a vendor thread). Surfaced only.
- **Waiting — no response yet** — a thread where **your** `SENT` message is the tail and
  it has gone unanswered longer than `email.awaiting_reply_days` (see "Awaiting-reply
  sweep"). Surfaced only.
- **Reply-state suppression:** a thread is **handled** and suppressed **only when your
  `SENT` message is the latest message AND it is not stale**. If new inbound arrived
  *after* your reply, surface it (New / Reminders / Updates as fits); if your reply is the
  tail but older than `awaiting_reply_days`, surface it under **Waiting**. A handled-and-
  fresh thread stays suppressed. (Still eligible for auto-extract if confirmation-grade.)
- "Actionable/relevant" = a direct ask/question to the user, a stated deadline, or a match
  to an **active** task/project/`email.watch` — but not confirmation-grade.

**Relevance signal:**
- **Free:** match against the task/project titles the caller **already** queried this
  run (no extra queries).
- **Opt-in:** `email.wiki_match: true` adds a per-candidate `notion-search` across
  Wiki/Raw. Off by default (costly / noisy).

**Reply-state detection:** whether **your `SENT` message is the thread tail** (the latest
message) — identity-agnostic, no hardcoding the user's address; read from per-message
`SENT` labels + message ordering. Checked only in stage 2, for read-and-plausibly-
important and awaiting-reply candidates (adds no cost to ignored/unread-only flows). A
`SENT` message that is *not* the tail — new inbound followed it — does **not** mark the
thread handled.

## Awaiting-reply sweep (stale outbound)

The incremental `after:<last_scan_ts>` window only sees **new inbound** — a thread you
sent days ago and are still waiting on has no message in-window, so it can never surface
there. A **separate bounded query** finds stale outbound and feeds the **Waiting — no
response yet** group:

- Query `in:sent newer_than:<email.awaiting_lookback_days> older_than:<email.awaiting_reply_days>`
  (defaults 30 / 5 days) — the look-back bound keeps it from scanning your whole Sent mail;
  the `older_than` bound is the "too long" staleness threshold.
- **Tail-check (stage 2, candidates only):** `get_thread` each candidate; surface it only
  when **your `SENT` message is the thread tail** (no inbound after it). Drop any thread
  that already got a reply.
- **Read-only, surfaced only** — never written, never labeled. Honors the same pagination
  cap (5 pages / 250 threads); report truncation rather than dropping silently.
- Runs in the full scan only; the single targeted-extract mode skips it.

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

Auto-extract fires **only for items whose triggering message is ≥ `last_scan_ts`** — the
same **new slice** the chat render marks `· new`. The day-scoped widening in "Scan window"
applies to the **read/render path only**; the **write path stays incremental**, so a
re-run later the same day re-*renders* the whole day but re-*writes* nothing. Dedup
therefore rests on the **write path being incremental** — **not** on the scan window
being near-empty (which the day-scoped `window_start` makes false). **No Gmail writes, no
Raw lookups.**

- **Trade-off:** a partial run that failed before advancing `last_scan_ts` re-scans the
  same window next time — the safe direction; the high auto-extract bar keeps any
  duplication negligible.
- **Mid-day-flip edge:** if `auto_extract` is flipped **on** mid-day, that run's
  confirmations note may label an earlier (pre-cursor) item "Extracted to Raw" when
  nothing was actually written. Rare, self-corrects on the next run — called out here
  rather than engineered away.

## Rendering the 📧 section

The **📧 From your inbox** section renders up to four groups — **New**, **Reminders —
you haven't replied**, **Updates / heads-up**, and **Waiting — no response yet** —
**omitting any group that is empty**. Below them, note any confirmations/extractions
(see below) and any degradations (window capped, epoch fallback, email tool absent,
window truncated).

**Two renderings from one classified set.** The window rule produces **one** surfaced
set per run (never two scans). Because `window_start` is a superset of both "today" and
"since `last_scan_ts`", that one set serves both surfaces — they differ only in a marker:

- **Journal 📧 section** — the **full set, unmarked**: a complete, self-contained record
  of the day (plus any gap-guard catch-up spillover, already flagged).
- **Chat brief** — the **same set**, with **`· new`** appended to any item whose
  triggering message timestamp is **≥ `last_scan_ts`**.
  - **Every item new** (ordinary first-run-of-the-day): **omit** the markers as noise.
  - **No item new** (a re-run after the cursor already advanced — the Finding 2 case):
    render the day's groups and add one line —
    **`Nothing new since your ⟨HH:MM in preferences.timezone⟩ scan.`**
- **Waiting** is **never marked** — the awaiting-reply sweep is a bounded `in:sent` query
  with no window; it is already current in both surfaces.

This is a **re-derived snapshot, not an append-only log.** A thread that surfaced in the
morning but is now handled (replied, extracted, ignored) legitimately **drops off** —
exactly like a completed task leaving the task list. "Nothing that appeared in the
morning is ever removed" is explicitly **not** a property.

**Per-email bullet** (mirrors `today`'s per-task bullet convention in
`notion-conventions.md`, but a Gmail thread is an external URL, not a Notion page —
so use a plain markdown link in **both** chat and the Journal body, never a
`<mention-page>`): `- [<cleaned subject>](<gmail thread link>) — <sender> · <one-line
why it surfaced>`. Example:
`- [Re: lease renewal terms](https://mail.google.com/…) — landlord · asks for a decision by Fri`.

**Confirmations / extractions note** (below the groups):
- When `email.auto_extract` is **off** (the default), a qualifying confirmation is
  **surfaced for on-demand extraction**, not written:
  `- Confirmation you may want to extract: [<subject>](<gmail link>)`.
- When `email.auto_extract` is **on**, an auto-written row is noted as
  `- Extracted to Raw: [<subject>](<gmail link>)` and, per the Referencing-pages-inline
  convention, may additionally link the created Raw row (`<mention-page>` in the Journal
  body, `[title](url)` in chat).
- The **Extracted to Raw** note reflects only the **new slice** (`≥ last_scan_ts`): the
  write path is incremental (see "Dedup"), so a re-run later the same day re-*renders* the
  whole day but does **not** re-note the morning's already-written confirmations. (The
  *surfaced* "Confirmation you may want to extract" line, by contrast, covers the whole
  day's set — surfacing is a read-path render.)

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
