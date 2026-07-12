---
name: today
description: Use when the user asks for their daily brief, what's due, or what's on today — reports overdue and due-today tasks across all task databases, today's calendar events, and upserts a daily journal entry. Runs unattended in Routines (reports only, never prompts).
---

# today

Daily brief: overdue and due-today tasks across every task database
(personal plus each registered shared space), today's calendar events if a
calendar tool is available, and an upserted Journal row for today. Designed
to run unattended in Routines — it never prompts, and when run
non-interactively it also writes the brief into today's Journal row so it
stays retrievable from Notion even if the session output is never seen.

Consult `../shared-references/schema.md` for the Task schema (Status, Due,
Assignee) and the Journal schema (Name, Date, Type), and the `config.json`
key set. Consult `../shared-references/notion-conventions.md` for MCP
quirks (single-source queries, dual-path fallback, concurrency-safe writes,
rate limits, async writes). Use the config keys and property names exactly
as `schema.md` defines them — do not paraphrase or rename.

## Behavior

### 1. Load config or discover

Same pattern as `capture`: look for `config.json` in the launch folder
first; if present and valid, read `tasks_personal.data_source_url`,
`shared_spaces[].tasks` (data source URLs), `journal.data_source_url`,
`preferences.timezone`, `preferences.calendar_tool`, and `notion_user_id`
from it.

If no `config.json` is found, fall back to discovery: `notion-search` for
the root page named exactly `Second Brain` (emoji prefix allowed), then
`notion-fetch` its `Home` child page and read the fenced ```json config
block for the same keys, per the discovery convention in `schema.md`.

If neither resolves (no config file and discovery finds no root/Home config
block, or finds multiple ambiguous candidates): **fail loudly**. State
plainly that the second brain isn't set up yet, point the user at the
`setup` skill, and **stop**. Never guess at IDs, never invent a target, and
never silently produce a partial brief from missing config.

Resolve "today" as a calendar date in `preferences.timezone` before issuing
any query — every `Due <=` comparison and the Journal `Date` match below use
this resolved date.

### 2. Query tasks — one data source at a time, merge in the skill layer

For `tasks_personal.data_source_url` and, separately, for each entry in
`shared_spaces[].tasks`, issue an independent **single-source**
`notion-query-data-sources` call. Never issue a cross-data-source query —
each call targets exactly one data source, per `notion-conventions.md`.

- **Personal:** filter `Due <= today` AND `Status != Done`.
- **Each shared space:** the same filter, plus `Assignee = me` (matched
  against `notion_user_id`) — a shared brief only ever surfaces tasks
  assigned to the current user, never a partner's.

Merge all results in this skill after every query returns — never ask
Notion to span data sources. Split the merged set into **Overdue**
(`Due < today`) and **Due today** (`Due = today`), and within each group,
group sensibly by source (personal, then each shared space by name) so the
brief reads as "what's overdue" / "what's due today" per space rather than
one flat list.

**Dual-path fallback:** if any `notion-query-data-sources` call returns an
upgrade/plan-gating prompt instead of results, fall back to scoped
`notion-search` + `notion-fetch` of that data source for the same task
list, and **note the degradation** in the brief (which DB fell back, and
that results may be less precisely filtered/sorted since the fallback path
can't apply the same structured filter).

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
for a row with `Date` = today (resolved date from §1).

- **No match:** create one with `notion-create-pages` — `Name` a short
  label for the date, `Date` = today, `Type` = `Daily`.
- **Match found:** update that row rather than creating a second one for
  the same date — prefer `update_content` / `insert_content` over a
  whole-page replacement, per `notion-conventions.md`, so concurrent human
  edits to the day's journal entry aren't clobbered.

Exactly one Journal row exists per date after this step, whether this is
the first run of the day or a re-run.

### 5. Produce the brief; Routine-safe

Always produce the brief as text in the response: Overdue tasks (grouped by
source), Due-today tasks (grouped by source), Calendar section (if
applicable, §3), and a note of any dual-path degradation (§2).

**Never prompt** — nothing in this skill is worth blocking on; every
ambiguity (missing calendar tool, degraded query path, empty task list) is
reported, not asked about. This must hold whether invoked interactively or
from a Routine.

**When run non-interactively** (e.g. from a Routine, with no one to read
the chat response): also write or append the brief text into today's
Journal row (§4) via `insert_content`, so the brief is retrievable from
Notion even though no one saw the session output. When run interactively,
this Journal write still happens (it's the same upsert either way) — the
distinction is only that non-interactive runs must not depend on the chat
response being read.

## Errors

- **MCP unavailable** (`notion-query-data-sources`, `notion-search`,
  `notion-fetch`, `notion-create-pages`, or `notion-update-page` fails,
  unauthenticated or otherwise): say so plainly, point at claude.ai →
  Settings → Connectors → Notion, and **stop** — never fake a brief or
  report tasks that weren't actually fetched.
- **Rate limiting (`429`):** honor `Retry-After` and back off before
  retrying, per `notion-conventions.md`.
- **Large/async writes:** if the Journal upsert write returns an async
  task, poll `notion-get-async-task` rather than assuming completion.
- **Partial failure** (e.g. one shared space's query fails but others
  succeed): report the brief with whatever succeeded, and clearly flag
  which source failed and why — don't silently drop it or fail the whole
  brief for one bad data source.

## Smoke test

Smoke test (run against a live Notion workspace):
- Seed one overdue task and one due-today task (personal); optionally one
  in a shared DB assigned to the current user.
- Run `today`.

Assertions: the brief lists overdue + due-today tasks across ALL task
databases (queried one at a time, merged), grouped sensibly; a Calendar
section appears IF a calendar tool is present in the session (otherwise
it's omitted, not an error); a Journal row for today exists afterward
(created if missing, updated if already present — one row per date); no
prompts are issued at any point (Routine-safe).
