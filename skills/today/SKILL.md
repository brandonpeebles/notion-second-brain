---
name: today
description: Use when the user asks for their daily brief, what's due, or what's on today — reports overdue and due-today tasks across all task databases, today's calendar events, and upserts a daily journal entry. Runs unattended in Routines (reports only, never prompts).
---

# today

Daily brief: overdue and due-today tasks across every task database
(personal plus each registered shared space), today's calendar events if a
calendar tool is available, and an upserted Journal row for today. Designed
to run unattended in Routines — it never prompts. On every run, interactive
or not, it also writes the brief into today's Journal row so it stays
retrievable from Notion even when the session output is never seen.

Consult `../shared-references/schema.md` for the Journal schema (Name, Date,
Type) and the `config.json` key set, and `../shared-references/task-db-mapping.md`
for how each task DB's real property names and status values are resolved.
Consult `../shared-references/notion-conventions.md` for MCP quirks
(single-source queries, dual-path fallback, concurrency-safe writes, rate
limits, async writes) and its **Referencing pages inline** convention (how
each task links to its source row — native mentions in the Journal body,
markdown links in chat), and `../shared-references/query-plan-gating.md` for
the plan-gate error signature and fallback decision tree. Use config keys
and the Journal's property names exactly as `schema.md` defines them — the
Journal is an internal DB with no mapping. Task-DB property names and status
values are **never** hard-coded: for every task DB (personal and each shared
space), resolve each role (`title`/`status`/`due`/`scheduled`/`assignee`/
`priority`/`project`/`source`) through that DB's own `properties` and
`status_values`, per `task-db-mapping.md`'s skill resolution rule — look up
the role, use the returned real name, and skip (never substitute a canonical
name) when a role isn't mapped for that DB.

## Behavior

### 1. Load config or discover

Same pattern as `capture`: look for `config.json` in the launch folder
first; if present and valid, read `tasks_personal.data_source_url`,
`shared_spaces[].tasks` (data source URLs), `journal.data_source_url`,
`preferences.timezone`, `preferences.calendar_tool`, and `notion_user_id`
from it.

If no `config.json` is found, fall back to discovery: `notion-search` for
the root page named exactly `Second Brain` (emoji prefix allowed), then
`notion-fetch` its `AGENTS` child page and read the fenced ```json config
block for the same keys, per the discovery convention in `schema.md`.

If neither resolves (no config file and discovery finds no root/AGENTS config
block, or finds multiple ambiguous candidates): **fail loudly**. State
plainly that the second brain isn't set up yet, point the user at the
`setup` skill, and **stop**. Never guess at IDs, never invent a target, and
never silently produce a partial brief from missing config.

Resolve "today" as a calendar date in `preferences.timezone` before issuing
any query — every date comparison and the Journal `Date` match below use
this resolved date.

**Unconfirmed-mapping guard:** for each task DB `today` will query (personal
and every shared space), check its `confirmed`/`unconfirmed_roles` marker.
If `status` is unconfirmed, skip that DB entirely (it can't be safely
filtered to "not done" at all). If `due` is unconfirmed, skip that DB for
the overdue/due-today buckets. If `scheduled` is unconfirmed, skip that DB
for the planned-today bucket only. If `assignee` is unconfirmed for a shared
DB, skip the "mine only" `⟨assignee⟩ = notion_user_id` filter for that DB —
never apply an unconfirmed mapping — and report it as **unconfirmed —
needs setup to confirm**, distinct from the unmapped-assignee case in §2. In
every case, report which DB was skipped, for which bucket, and that `setup`
must confirm that DB's mapping — this is a per-DB skip, not a hard stop for
the whole brief; DBs with confirmed mappings still produce their part of the
brief.

**Unresolved personal-DB guard:** if `tasks_personal` is the
`{"pending_selection": true, "candidates": [...]}` shape (no
`data_source_url` — per `task-db-mapping.md`) rather than the normal mapping
shape, skip the personal task DB entirely for this run — it can't be queried
without a resolved data source — and report it as **unresolved — needs
setup to resolve the pending personal task DB pick**, the same per-DB-skip
pattern as the unconfirmed-mapping case above, one level up (an unresolved DB
rather than a resolved DB with an ambiguous role). Shared spaces are
unaffected — every shared space still produces its part of the brief, and
`shared_spaces[].tasks` has no `pending_selection` variant.

### 2. Query tasks — one data source at a time, merge in the skill layer

For `tasks_personal.data_source_url` and, separately, for each entry in
`shared_spaces[]` (using that entry's `.tasks.data_source_url`), issue an
independent **single-source** `notion-query-data-sources` call. Never issue a cross-data-source query —
each call targets exactly one data source, per `notion-conventions.md`.

For each task DB, resolve its own `properties.due`, `properties.scheduled`,
`properties.status`, `status_values.done[]`, `properties.project`, and (for
shared spaces) `properties.assignee` — per `task-db-mapping.md` — before
building filters. Never assume any two DBs share property names.

For every row returned, **retain its page id/URL** (available on both the
structured and fallback paths — see the **Referencing pages inline**
convention in `notion-conventions.md`) alongside its mapped field values;
§5 needs it to link each task back to its source row.

- **Personal:** filter `⟨due⟩ ≤ today AND status ∉ status_values.done[]`,
  using this DB's own `properties.due` and `properties.status`.
- **Each shared space:** the same filter, plus, when `properties.assignee`
  is mapped **and confirmed**, `⟨assignee⟩ = notion_user_id` — a shared brief
  only ever surfaces tasks assigned to the current user, never a partner's.
  If a shared DB has **no** `assignee` mapping, it can't be narrowed to "mine
  only" — do not silently include everyone's tasks or silently drop the DB;
  query it unfiltered by assignee and clearly label that source's section in
  the brief as **unfilterable by assignee (shows all tasks in this DB)** —
  this DB doesn't track assignee. If a shared DB's `assignee` mapping is
  present but listed in `unconfirmed_roles` (per the Unconfirmed-mapping
  guard in step 1), do not apply the filter using that unconfirmed
  mapping — query it unfiltered by assignee and label that source's section
  as **unconfirmed — needs setup to confirm**, not unfilterable.
- **A DB with no `due` mapping at all** is omitted from the overdue/due-today
  buckets entirely — note this omission in the brief (a reportable omission,
  not an error), the same way §3 notes an omitted Calendar section.
- **Planned today:** for any DB whose `properties.scheduled` is mapped,
  issue an additional `⟨scheduled⟩ = today` filter and surface those results
  in a third **Planned today** bucket, alongside Overdue and Due today. Not
  every DB maps `scheduled` — e.g. a personal Tasks DB that only maps `due`
  simply contributes nothing to this bucket, with no error and no note (an
  unmapped optional role is normal, not a degradation).

Merge all results in this skill after every query returns — never ask
Notion to span data sources. Split the merged set into **Overdue**
(`⟨due⟩ < today`), **Due today** (`⟨due⟩ = today`), and **Planned today**
(`⟨scheduled⟩ = today`, only from DBs that map `scheduled`), and within each
group, group sensibly by source (personal, then each shared space by name)
so the brief reads as "what's overdue" / "what's due today" / "what's
planned today" per space rather than one flat list.

**Dual-path fallback:** if any `notion-query-data-sources` call **throws the
plan-gate error** (the `400` signature in `query-plan-gating.md`, not a
generic 400 from a bad filter), fall back to scoped `notion-search` +
`notion-fetch` of that data source for the same task list, and **note the
degradation** in the brief (which DB fell back, and that results may be less
precisely filtered/sorted since the fallback path can't apply the same
structured filter).

### 3. Calendar — auto-detect, omit if none

Auto-detect the calendar source: if `preferences.calendar_tool` is set in
config, use that tool; otherwise use whatever calendar tool the current
session exposes. List today's events (resolved date from §1) and include
them as a Calendar section in the brief.

If no calendar tool is set in config and none is available in the session,
**omit the Calendar section entirely** — do not error, and do not report
its absence as a failure; a brief with no calendar section is a normal,
complete brief.

### 4. Journal upsert — one row per date

Query `journal.data_source_url` (single-source `notion-query-data-sources`)
for a row matching **`Date` = today AND `Type` = `Daily`** (resolved date
from §1). Match on both — never on `Date` alone: other skills (e.g.
`triage`) file their own dated Journal rows with `Type` left unset, and
matching by date alone would collide with them.

- **No match:** create one with `notion-create-pages` — `Name` a short
  label for the date, `Date` = today, `Type` = `Daily`.
- **Match found:** update that single `Type = Daily` row rather than
  creating a second one for the same date — prefer `update_content` /
  `insert_content` over a whole-page replacement, per
  `notion-conventions.md`, so concurrent human edits to the day's journal
  entry aren't clobbered. (This row upsert is unconditional; the brief
  **content** is always written/updated into the row per §5.)

Then `notion-fetch` the matched (or newly created) row's page id to read its
current body markdown — §5 needs the exact current text of any existing
`## Daily Brief — ⟨date⟩` section to replace it in place.

Exactly one `Type = Daily` Journal row exists per date after this step,
whether this is the first run of the day or a re-run. Other rows sharing
that date (e.g. `triage`-promoted notes) are untouched — the match is
Type-scoped.

### 5. Produce the brief; Routine-safe

Always produce the brief as text in the response: Overdue tasks (grouped by
source), Due-today tasks (grouped by source), Planned-today tasks (grouped
by source, only from DBs mapping `scheduled`), Calendar section (if
applicable, §3), and a note of any dual-path degradation, mapping-driven
omissions, or unconfirmed-mapping skips (§2).

**Per-task bullet.** Each task is one bullet that links back to its source
row and carries light context — not a bare title. Two renderings, per the
**Referencing pages inline** convention in `notion-conventions.md` (chat and
Journal body differ; never emit `<mention-*>` tags to chat):

- **References its source row** using the row's retained URL (§2) — Journal
  body: `<mention-page url="⟨row url⟩"/>`; chat: `[⟨mapped title⟩](⟨row url⟩)`.
- **Then appends context, each segment only when that DB maps the role** —
  skip an unmapped role, never substitute a canonical name (same mapping rule
  as everywhere else):
  - **Due date** — Journal body: `<mention-date start="⟨due⟩"/>`; chat: a
    human date — immediately followed by a computed **urgency label** from
    `⟨due⟩` vs the §1 resolved today: `(⟨N⟩d overdue)` in the Overdue bucket,
    `(due today)` in Due today, `(in ⟨N⟩d)` in Planned today.
  - **Project** — the mapped `project` value when `project` is mapped for
    that DB; omit the whole segment otherwise.

Keep the existing grouping (bucket → source/space) and sort within each
bucket by due date, most overdue first. Example (Journal body):
`- <mention-page url="…"/> — due <mention-date start="2026-07-09"/> (3d overdue) · Home Reno`;
same task in chat:
`- [Fix the sink](https://www.notion.so/…) — due Jul 9 (3d overdue) · Home Reno`.

**Never prompt** — nothing in this skill is worth blocking on; every
ambiguity (missing calendar tool, degraded query path, empty task list) is
reported, not asked about. This must hold whether invoked interactively or
from a Routine.

The §4 Journal **row** upsert (create/update the row with `Date` = today
and `Type` = `Daily`, one row per date) is **unconditional** — it happens
on every run, interactive or not.

Writing the brief **content** into that row's body is also **unconditional**
— every run, interactive or not, so the brief stays retrievable from Notion
even when no one saw the session output. Write it as one delimited section
headed by a stable marker, `## Daily Brief — ⟨date⟩` (resolved date from
§1), so re-runs **refresh that one section rather than appending a duplicate
copy** and repeated runs in a day never bloat the row:

- **No such section in the fetched body (first run of the day):** append it
  with `insert_content` at `{"type":"end"}`.
- **Section already present:** replace it in place with `update_content` —
  `content_updates: [{old_str: ⟨the exact existing section text, heading
  through its end, from the §4 body fetch⟩, new_str: ⟨the regenerated
  section⟩}]`. Because §4 fetched the current body immediately before this
  write, the `old_str` exact-match holds — including the mention/date-pill
  bullets, which the fetch returns in *resolved* form, so `old_str` must be
  that fetched text, never the markdown you originally wrote (per the
  Referencing pages inline idempotency caveat in `notion-conventions.md`).
  Content **outside** the section
  (human edits, `triage`-promoted notes) is untouched — this is the
  concurrency-safe path from `notion-conventions.md`, not a whole-page
  replacement.
- **If the in-place replace fails to match** (e.g. the section was
  hand-edited between fetch and write): fall back to appending a fresh
  section with `insert_content` rather than erroring — persistence beats
  perfect dedup, and the next run re-converges to a single section.

## Errors

- **MCP unavailable** (`notion-query-data-sources`, `notion-search`,
  `notion-fetch`, `notion-create-pages`, or `notion-update-page` fails,
  unauthenticated or otherwise): say so plainly, check `/mcp` and confirm
  you're logged in with your claude.ai account, and **stop** — never fake a
  brief or report tasks that weren't actually fetched.
- **Rate limiting (`429`):** honor `Retry-After` and back off before
  retrying, per `notion-conventions.md`.
- **Large/async writes:** if the Journal upsert write returns an async
  task, poll `notion-get-async-task` rather than assuming completion.
- **Partial failure** (e.g. one shared space's query fails but others
  succeed): report the brief with whatever succeeded, and clearly flag
  which source failed and why — don't silently drop it or fail the whole
  brief for one bad data source.
- **Unconfirmed mapping** (a task DB's `status`, `due`, `scheduled`, or
  `assignee` role is listed under `unconfirmed_roles`): this is the same "partial failure,
  report which source failed" pattern above, scoped to one DB and/or one
  bucket rather than the whole brief — skip only the affected DB/bucket (per
  the §1 guard), report that `setup` must confirm the mapping, and still
  produce the rest of the brief from DBs with confirmed mappings.

## Smoke test

Smoke test (run against a live Notion workspace):
- Seed one overdue task and one due-today task (personal, using that DB's
  own mapped due-role property and open status value); give one a mapped
  `project` value and leave the other's project unset (to prove the skip).
  Optionally one in a shared DB assigned to the current user, and, if that
  shared DB maps `scheduled`, one task scheduled for today.
- Run `today`.

Assertions: the brief lists overdue + due-today tasks across ALL task
databases (queried one at a time, merged, each filtered through its own
mapping — no literal `Due`/`Status`/`Done`/`Assignee` assumed), grouped
sensibly; a Planned-today section appears for any DB that maps `scheduled`
and is silently absent from DBs that don't; a Calendar section appears IF a
calendar tool is present in the session (otherwise it's omitted, not an
error); a DB with an unconfirmed `due`/`status`/`scheduled` role is skipped
for the affected bucket and reported, not silently included or excluded; a
Journal row for today exists afterward (created if missing, updated if
already present — one `Type = Daily` row per date, using the Journal's
canonical `Date`/`Type` properties, unaffected by task-DB mapping) and its
body contains the brief under a `## Daily Brief — ⟨date⟩` marker section;
**each task bullet in the Journal body is a `<mention-page>` reference to its
source row** (renders the task's icon + live title, and a backlink to the
Journal row now appears on the referenced task), with a `<mention-date>` due
pill and the correct urgency label (`(Nd overdue)` / `(due today)` /
`(in Nd)`), and the project segment present only for the task that maps/sets
it and absent otherwise (no canonical fallback); the **chat output** renders
the same tasks as `[title](url)` markdown links, not raw `<mention-*>` tags;
after a **second run the same day** the row still has exactly one Daily Brief
section (refreshed in place, not duplicated — the mention/date-pill bullets
don't break the in-place match) and any body content outside that section is
unchanged; no prompts are issued at any point (Routine-safe).
