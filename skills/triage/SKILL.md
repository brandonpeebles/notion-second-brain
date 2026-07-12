---
name: triage
description: Use when the user wants to process or sweep the Inbox, or run a weekly review — proposes promoting each captured item to a task, wiki page, or journal entry, or archiving it, then executes the batch after a single confirmation.
---

# triage

Sweep the Inbox: for each captured row, propose promoting it to a Task, a
Wiki page, or a Journal entry — or archiving it (discard) — or leaving it in
the Inbox as explicitly reviewed-but-not-ready (`Kept`). Present the full
proposed mapping and get **one** confirmation, then execute the whole batch.
v0.1 promotes to the wiki inline (no separate `ingestor` agent — that lands
in v0.2).

Consult `../shared-references/schema.md` for the Inbox, Task, Journal, and
Archive schemas (property names/types), the Wiki rules, and the
`config.json` key set. Consult `../shared-references/two-person-rules.md`
for shared-vs-personal Task routing and assignee resolution. Consult
`../shared-references/notion-conventions.md` for MCP quirks (single-source
queries, the wiki parent-URL quirk, no page-trash tool, rate limits, async
writes). Use the config keys and property names exactly as `schema.md`
defines them — do not paraphrase or rename.

## Behavior

### 1. Load config or discover

Same pattern as `capture` and `today`: look for `config.json` in the launch
folder first. If present and valid, read `inbox.data_source_url`,
`tasks_personal.data_source_url`, `shared_spaces[].tasks` (and
`shared_spaces[].members`), `journal.data_source_url`,
`archive.data_source_url`, `wiki.data_source_url` and the wiki page URL
under `wiki`, `home_page`, and `preferences.timezone` from it.

If no `config.json` is found, fall back to discovery: `notion-search` for
the root page named exactly `Second Brain` (emoji prefix allowed), then
`notion-fetch` its `Home` child page and read the fenced ```json config
block for the same keys, per the discovery convention in `schema.md`.

If neither resolves: **fail loudly**. State plainly that the second brain
isn't set up yet, point the user at the `setup` skill, and **stop**. Never
guess at IDs, never invent a target, never assume a filesystem exists beyond
this one optional config-file check.

### 2. Read the Inbox — single-source query

Issue one **single-source** `notion-query-data-sources` call against
`inbox.data_source_url`, filtered to `Triage = New`. This is the working set
for a normal triage sweep. Never issue a cross-data-source query — the
Inbox is the only data source read here (Tasks/Wiki/Journal/Archive are
write-only targets in this skill, per `notion-conventions.md`).

**Dual-path fallback:** if the query returns an upgrade/plan-gating prompt
instead of results, fall back to scoped `notion-search` + `notion-fetch` of
the Inbox data source, and note the degradation in the output.

If the working set is empty, say so and stop — nothing to triage.

### 3. Propose a destination for each row

For every row in the working set, propose exactly one of:

- **Task** — imperative/actionable text ("call the dentist", "schedule X").
  Route personal vs. a named shared space per `two-person-rules.md`: shared
  only when that space is named or clearly implied by the row's `Name` /
  `Source`; otherwise personal. If routed to a shared space, resolve an
  `Assignee` from that space's `members`; if none is named, flag it in the
  proposal rather than silently omitting it (see §5, §6).
- **Wiki** — durable reference/knowledge (how-tos, standing information,
  things worth finding again) rather than a dated note.
- **Journal** — a dated note/reflection tied to when it was captured, not
  durable reference material.
- **Archive** — junk, duplicates, or anything not worth keeping (discard).
- **Keep** — reviewed but not ready to decide (needs more info, an idea
  still forming). No page is created; the row stays in the Inbox with
  `Triage = Kept` so it resurfaces in a future sweep or the weekly review
  (§7) instead of silently re-appearing as `New`.

Use `Type` (the capturer's guess: Task/Note/Idea/Reference) and `Source` as
signals, but the proposal is triage's own judgment, not a mechanical copy of
`Type` — a `Note` can still be Task-like ("note: schedule dentist"), and a
`Task`-typed row can turn out to be reference material.

### 4. Batch-confirm — exactly once

Present the **full** proposed mapping as a single list — one line per row:
its current `Name`, the proposed destination, and any destination-specific
detail (shared vs. personal + assignee for Task; discard for Archive; etc.)
— and get **one** confirmation before writing anything. Do not confirm
row-by-row and do not write before the confirmation covers the whole batch:
a single confirmation authorizes the entire batch of writes.

If the user adjusts individual rows in their reply, fold the adjustments in
and treat that reply as the confirmation for the (now-adjusted) batch — do
not loop back for a second full confirmation unless the user asks a
clarifying question first.

**Routine mode (non-interactive):** there is no one to confirm with. Do
**not** execute and do **not** prompt. Instead, create a report page (a
child page under `home_page`) containing the same proposed mapping the
interactive path would have presented, and stop — nothing is written to
Task/Wiki/Journal/Archive, and no Inbox row's `Triage` changes. This mirrors
`today`'s non-interactive persistence pattern: the output must survive even
though no one is reading the chat response.

### 5. Execute the confirmed batch

Once confirmed (interactive path only — never in Routine mode, per §4),
write each row to its destination.

**Where each `Triage` state comes from** (`schema.md`'s enum is
`New / Kept / Promoted`): a Task, Wiki, or Journal promotion sets the Inbox
row's `Triage = Promoted`; the "Keep in Inbox" outcome sets `Triage = Kept`
(reviewed, deliberately left in the Inbox to revisit); an untriaged row
stays `Triage = New`. (The brief attaches "Promoted/Kept" to the Journal
step; here `Kept` is factored out into its own Keep outcome so a
Journal-promoted note is unambiguously `Promoted` and every `Kept` row is a
reviewed-and-left-in-Inbox row — the two states the weekly review, §7, then
counts and ages.)

- **Task** → `notion-create-pages` in the target Tasks data source
  (`tasks_personal.data_source_url` or the named
  `shared_spaces[].tasks.data_source_url`), per `schema.md`: `Name` from the
  Inbox row; `Status = Not started`; `Due` if a date is present/parseable in
  the row text (against `preferences.timezone`), else unset; `Source`
  carried from the Inbox row's `Source`; `Assignee` set when routed to a
  shared space (§3/§6). Then `notion-update-page` the Inbox row:
  `Triage = Promoted`.
- **Wiki** → `notion-create-pages` parented to the **wiki page URL** from
  config — never the wiki's data source URL, per the wiki parent-URL quirk
  in `notion-conventions.md`. Title from the Inbox row's `Name`; body seeded
  from the row's content/`Source`. Do not attempt to set custom wiki
  properties (e.g. Tags) via MCP — they're human-only. Then
  `notion-update-page` the Inbox row: `Triage = Promoted`.
- **Journal** → **always** `notion-create-pages` a **distinct new row** in
  `journal.data_source_url` for the promoted item: `Name` from the Inbox
  row, `Date` = the Inbox row's `Captured` date, body seeded from the row's
  content/`Source`, and **leave `Type` unset** (do not set `Type = Daily`).
  Never match-and-insert into an existing Journal row. The invariant: `today`
  owns exactly the one `Type = Daily` row per date and finds it by matching
  `Date + Type = Daily`; triage-promoted notes are separate dated rows with
  `Type` left unset, so `today`'s Type-scoped upsert can never match them and
  the two skills can never clobber each other's Journal content. (A dedicated
  note `Type` arrives with the v0.3 `fold` schema extension; until then,
  unset is correct.) Then `notion-update-page` the Inbox row:
  `Triage = Promoted`.
- **Archive (discard)** → `notion-move-pages` the Inbox row into
  `archive.data_source_url` — there is no page-trash tool, so the row must
  physically leave the Inbox, not just get flagged. `Archived` is automatic
  (`created_time`) — never set it manually. After the move, set
  `Origin = "Inbox"` via `notion-update-page` if it isn't already populated.
- **Keep** → no page created, no move. `notion-update-page` the Inbox row:
  `Triage = Kept`.

Execute writes per row; if one row's write fails mid-batch, continue with
the rest and report exactly which rows succeeded and which failed (§8) —
don't silently drop a failure or abort the whole batch for one bad row.

### 6. Assignee gap on shared Task rows

Per `two-person-rules.md`, a shared-space task with no assignee is
incomplete. If a row is routed to a shared space and no partner is named:
in the batch-confirm step (§4), surface the gap and ask before writing (same
as `capture`). Routine mode never reaches this — §4 diverts to a report page
before any write, so the gap is simply visible in that report for the user
to resolve on the next interactive run.

### 7. Weekly-review mode

When asked for a weekly review (or similar periodic-review framing) instead
of a plain sweep: query the **full** Inbox (not filtered to `Triage = New`)
via the same single-source `notion-query-data-sources` call, and produce a
summary in addition to (or instead of) the normal §3 proposals:

- Counts by `Triage` (`New` / `Kept` / `Promoted`).
- A called-out list of **stale** rows — `Triage = New` or `Kept` with
  `Captured` older than 14 days — since these are the ones quietly aging in
  the Inbox without a decision.

This summary is read-only and needs no confirmation. If the review also
proposes fresh triage for the `New` rows, that proposal still goes through
the normal batch-confirm (§4) before anything is written.

### 8. Errors

- **MCP unavailable** (`notion-query-data-sources`, `notion-search`,
  `notion-fetch`, `notion-create-pages`, `notion-update-page`, or
  `notion-move-pages` fails, unauthenticated or otherwise): say so plainly,
  point the user at claude.ai → Settings → Connectors → Notion, and
  **stop** — never fake a proposal, a confirmation, or a write.
- **Rate limiting (`429`):** honor `Retry-After` and back off before
  retrying, per `notion-conventions.md`.
- **Large/async writes:** if a create/move/update returns an async task,
  poll `notion-get-async-task` rather than assuming completion before
  reporting a row as done.
- **Partial batch failure:** report which rows succeeded and which failed
  (§5) — don't silently drop a failure or claim the whole batch succeeded
  when part of it didn't.

## Smoke test

Smoke test (run against a live Notion workspace):
- Seed Inbox: (a) "schedule dentist" [task-like], (b) "idea: weekend trip to
  the coast" [note/idea], (c) "test junk row" [discard].
- Run `triage`.

Assertions: triage proposes a batch — (a) → Task, (b) → Wiki or Journal,
(c) → Archive — presents it as a single mapping, and gets exactly **one**
confirmation. After confirming:
- (a) a Task row exists in the appropriate Tasks database with the expected
  properties, and the Inbox row's `Triage = Promoted`.
- (b) a Wiki page (parented to the wiki **page** URL, not its data source)
  or a Journal row exists, and the Inbox row's `Triage = Promoted`.
- (c) the row has physically **moved** into the Archive database (not just
  flagged) and no longer appears in the Inbox.
