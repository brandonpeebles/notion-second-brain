---
name: setup
description: Use when setting up or repairing the Notion second brain — discovers, scaffolds, and adopts the workspace structure, writes config.json, verifies the Notion connector, runs the query-gating test, and registers shared spaces. Idempotent and additive; safe to re-run to add spaces.
---

# setup

Foundational skill for the Notion second brain. Discovers or scaffolds the
workspace structure, adopts what already exists, writes `config.json`,
verifies the Notion connector, runs the query-gating test, and walks the user
through up to two UI-only manual steps (wiki turn-into-wiki, and the Waiting
status option — only when a personal task DB is newly scaffolded). Its successful run produces the live
workspace every other skill (`capture`, `today`, `triage`, `query`) operates
on. Idempotent and additive: re-running never destroys anything, and can be
used to add more shared spaces later.

Consult `../shared-references/schema.md` for the canonical DB schemas,
property names/types, the `config.json` key set, adopt/patch rules, and the
discovery config-block format. Consult `../shared-references/task-db-mapping.md`
for the task-DB mapping contract — role catalog, `status_values`, the
confirmed/unconfirmed non-interactive contract, and the derivation heuristics
this skill executes in §3b/§7. Consult `../shared-references/notion-conventions.md`
for MCP quirks (wiki creation, status defaults, no-trash, single-source
queries, dual-path fallback, rate limits, async writes, connector identity),
and `../shared-references/query-plan-gating.md` for the plan-gate error
signature and per-tier gate map (used by the query-gating test, §8).
`config.json`'s **keys** and the **internal DBs'** (Inbox/Journal/Archive)
property names stay canonical — use them exactly as `schema.md` defines them,
do not paraphrase or rename. **Task-DB property names and status values are
not canonical** — they are discovered per-DB and recorded as a mapping
(`properties`/`status_values`/`confirmed`) per `task-db-mapping.md`; never
force a task DB to match the scaffold shape.

## Behavior

Perform these steps in order. Do not skip ahead if an earlier step fails —
stop and report instead.

### 1. Verify connector

Call `notion-get-users`. On failure (no connector, unauthenticated, or any
MCP error): state plainly that the Notion connector is unavailable or
unauthenticated, point the user at claude.ai → Settings → Connectors →
Notion to connect it, and **stop**. Never fake success or proceed as if the
workspace is reachable.

On success, record the current user's id and display name — these become
`notion_user_id` and `user` in `config.json` (§9).

### 2. Locate or create the root

Look for `config.json` in the launch folder first. If it exists and is
valid, treat it as authoritative and skip straight to verifying its
contents against live Notion in step 3 (adopt path).

If no `config.json` is found, `notion-search` for a page named exactly
`Second Brain` (an emoji prefix such as `🧠` is allowed) that is private to
the current user, per the discovery convention in `schema.md`. If a root is
found this way, also `notion-fetch` its `🏠 Home` page for the fenced config
block (§6) — it may be the only surviving mapping record (see below).

- **Exactly one match:** adopt it as the root.
- **No match:** offer to create `🧠 Second Brain` (`notion-create-pages`),
  private to the current user, and use it as the root.
- **Multiple matches:** stop and ask the user which one to use. Never guess.

**Read the stored task-DB mapping on re-run.** Whenever a config source
exists — the launch-folder `config.json`, or (absent that) the Home page's
config block — load each task DB's `properties`/`status_values`/`confirmed`
from it before doing any discovery. A **confirmed** mapping is authoritative:
re-verify the DB still has those properties (a cheap `notion-fetch` schema
check), but **do not re-disambiguate** anything about it. Only run discovery
(§3b for the personal DB, §7 for shared spaces) for a task DB with **no**
stored mapping, or one recorded with `unconfirmed_roles`. This matters most
for cross-session Cowork use, where there's no durable `config.json` — the
Home block is the only place the mapping survives between sessions, and
reading it there is what prevents re-prompting the full disambiguation flow
on every run.

### 3. Scaffold or adopt the internal databases (Inbox, Journal, Archive)

Under the root, scaffold or adopt Inbox, Journal, and Archive using the
schemas in `schema.md` and its internal-DB adopt/patch rules. These three are
plugin-owned and stay canonical — unlike task DBs (§3b, §7), they are safe to
additively patch.

- For each database, `notion-search`/`notion-fetch` under the root for a
  child matching the display-name convention (`⚡ Inbox`, `📓 Journal`,
  `🗑 Archive`).
- **Missing:** create it with `notion-create-database` using the exact
  properties and types from `schema.md`.
- **Existing:** verify every required property exists with the right type;
  **add** any missing properties with `notion-update-data-source` (DDL).
  Never drop or retype an existing property.
- **Type conflict** (a required property exists with the wrong type): do
  **not** mutate it. Report the conflict for that database and skip it —
  continue with the remaining databases.

Record each database's `data_source_url` for `config.json`.

### 3b. Discover or scaffold the personal task DB (adopt-first)

The personal task DB (`tasks_personal`) is **not** forced into the scaffold
shape. Follow `task-db-mapping.md`'s adopt-never-mutate contract:

If step 2 loaded a **confirmed** mapping for `tasks_personal`, re-verify its
mapped properties still exist (`notion-fetch` the schema) and skip the rest
of this section. If it loaded an **unconfirmed** mapping, resume
disambiguation on just the `unconfirmed_roles` rather than starting over.

Otherwise, discover from scratch:

1. **Search for candidates.** `notion-search` the workspace for databases
   that look like a task DB (a title property plus a status- or
   select-typed property; display name matching `✅ Tasks`/"Tasks"/"To-do"
   is a strong signal but not required). Present the candidates found and
   ask the user which one (if any) is their personal task DB. In
   non-interactive mode: **zero candidates found** → proceed to the scaffold
   branch below (step 3); **exactly one obvious candidate** → adopt it (step
   2); **two or more ambiguous candidates** (more than one plausible match,
   none obviously "the one") → do **not** scaffold and do **not** guess —
   leave `tasks_personal` unresolved for this run, record the candidate list
   as a pending pick (mirroring how ambiguous *roles* are recorded under
   `unconfirmed_roles` below, and the wiki/Waiting "pending manual step"
   pattern in §10), and report it. Scaffolding a new DB only happens when no
   candidate exists at all.
2. **Adopt + map the chosen candidate** (no DDL, ever, against this DB):
   - Fetch its data source schema.
   - Run **role inference** (heuristics below) to identify `title`,
     `status`, and any of `due`/`scheduled`/`assignee`/`priority`/`project`/
     `source` present.
   - Derive `status_values` (`open_default`, `done[]`) from the status
     property's groups.
   - **Interactive:** echo the inferred mapping and `status_values` back to
     the user; let them correct any ambiguous role; once confirmed, record
     `"confirmed": true`.
   - **Non-interactive (Routine):** run inference, record the mapping with
     any ambiguous roles listed under `unconfirmed_roles`, and report them
     as pending picks — mirror the wiki/Waiting "pending manual step"
     pattern (§10). **Never guess** an ambiguous role.
   - If `title` or `status` cannot be identified at all (no candidate
     property), fail loudly for this DB and stop — per `task-db-mapping.md`'s
     fail-loudly rule.
3. **If the user has no existing personal task DB**, scaffold `✅ Tasks`
   using the scaffold-default shape from `schema.md` (`notion-create-database`)
   and give it an **identity mapping** — `properties` naming exactly the
   properties just created (`title:"Name"`, `status:"Status"`, `due:"Due"`,
   `assignee:"Assignee"`, `priority:"Priority"`, `project:"Project"`,
   `source:"Source"`) and `status_values` from Notion's default groups
   (`open_default:"Not started"`, `done:["Done"]`). An identity mapping is
   `confirmed: true` by construction — there's nothing to disambiguate in a
   shape setup just created.

Record `tasks_personal`'s `data_source_url` plus its `properties`/
`status_values`/`confirmed` (or `unconfirmed_roles`) for `config.json` (§9).

**Role-inference heuristics** (full contract and worked examples in
`task-db-mapping.md` — summarized here for execution):

- `title`-typed property → `title` (always unambiguous, exactly one exists).
- `status`-typed property → `status`. If none exists but a select property is
  named like "Status"/"State", it's a `status` candidate — ask which select
  is the status driver (Routine: mark `status` unconfirmed).
- Date properties → `due`/`scheduled`. One date property → `due`. Multiple →
  **ask which is the hard deadline (`due`) vs. the do/work date
  (`scheduled`)** (Routine: mark both unconfirmed).
- Person property → `assignee` (ask if multiple exist).
- Properties named/typed like "Priority"/"Project"/"Source" → `priority`/
  `project`/`source` by name+type match; omit the role entirely if no match.
- `status_values`: if `status` is Notion's native `status` type, use its
  built-in groups — To-do group → `open_default` (ask if more than one
  To-do value exists), Complete group → `done[]`. If `status` was filled by
  a plain select (no groups), ask the user which values are "done" and which
  is the open default. Always echo the derived `status_values` and confirm
  before writing (Routine: auto-infer, flag ambiguous grouping as
  `unconfirmed_roles`).

### 4. Wiki (guided, UI-only)

`notion-search`/`notion-fetch` under the root for a wiki. Notion wikis
cannot be created via MCP — "Turn into wiki" is UI-only (see
`notion-conventions.md`).

If no wiki is found, walk the user through it:

1. Create a page named `📚 Wiki` under the root (you may create this page
   via `notion-create-pages`, or ask the user to create it).
2. Ask the user to open it in Notion and choose "Turn into wiki" from the
   page menu.
3. Ask the user to paste the resulting page URL back.

Once you have the URL, adopt it: fetch it to confirm it's now a wiki, and
record `wiki.database_id` (the wiki page — the parent target other skills
use when creating new wiki pages) and `wiki.data_source_url`
(`collection://…`, used only for structured queries against the wiki) for
`config.json`. Remind the user (once, not per-run) that new wiki pages must
be parented to the wiki **page** URL, not its data source, and that custom
properties like Tags are human-only — MCP can only set Verification.

If a wiki is already found, adopt it directly (no manual step needed) and
skip to step 5.

In non-interactive/Routine mode (see §10), do not prompt for this step —
report it as a pending manual step instead.

### 5. Waiting status option (guided, UI-only — scaffolded personal DB only)

This step applies **only when setup scaffolded** the personal task DB in
§3b (step 3, no pre-existing DB adopted). Never add a status option to a DB
you don't own — an adopted DB (personal or shared) is handled by the
inference branch below instead.

For a **scaffolded** DB: check whether its Status property already has a
**Waiting** option by fetching the Tasks data source schema. If it's
missing, instruct the user to do a one-time manual UI step: open the Tasks
database in Notion, edit the Status property, and add a **Waiting** option
to the **In progress** group. MCP cannot add or rename status options —
Notion only allows creating the three defaults (Not started / In progress /
Done) via DDL (`notion-conventions.md`). After the user confirms, re-fetch
the Tasks data source schema, verify **Waiting** is now present, and record
it as `status_values.waiting`. Do not assume success from the user's word
alone — verify. In non-interactive/Routine mode, do not prompt; report this
as a pending manual step instead.

For an **adopted** personal or shared task DB: never add a status option.
Instead, **infer** a waiting/blocked value from the Status property's groups
during role inference (§3b/§7) — e.g. an In-progress-group value like
"Awaiting response" — and record it as `status_values.waiting`. If no such
value exists, leave `waiting` unmapped and let dependent behavior (e.g.
"blocked on partner" in `two-person-rules.md`) degrade gracefully.

### 6. Home page + config block

Under the root, `notion-search`/`notion-fetch` for `🏠 Home`. Create it with
`notion-create-pages` if missing; adopt it if present. Record `home_page`
for `config.json`.

Optionally script dashboard views with `notion-create-view` — e.g. a Tasks
board grouped by Status.

Write the fenced ```json config block into Home's body using
`insert_content` (concurrency-safe append/insert), never a whole-page
replacement — this avoids clobbering concurrent human edits per
`notion-conventions.md`. The block's keys and structure must mirror
`config.json` exactly (§9). If a config block already exists in Home, update
it in place rather than duplicating it.

### 7. Shared spaces (zero or more)

Ask the user whether to register any shared spaces now (e.g. a space shared
with a partner). This step is optional and additive — zero spaces is a
valid outcome, and re-running `setup` later can register more.

For each named shared space:

1. Locate its task database (search under the space's root, or ask the user
   for it) — the display name and shape are whatever already exists there;
   don't expect `✅ Tasks`'s scaffold shape.
2. **Discover and map it — never mutate.** If step 2 already loaded a
   confirmed mapping for this space's task DB, re-verify its properties
   still exist and skip to step 3. Otherwise run the same role-inference and
   `status_values`-derivation heuristics as §3b against this DB's schema:
   identify `title`/`status` (no candidate property at all for either → fail
   loudly, per §3b's rule; a candidate found but ambiguous → `unconfirmed_roles`,
   not a failure) and any optional roles present, derive `status_values` from its Status
   groups, echo and confirm the mapping (interactive), or record ambiguous
   roles under `unconfirmed_roles` and report them as pending picks
   (non-interactive/Routine — never guess). This DB is never scaffolded and
   never receives DDL — no `notion-update-data-source` call against it,
   ever, regardless of what properties it's missing relative to the
   scaffold shape.
3. Populate `members` by resolving each named member via `notion-get-users`.
4. Append the space to `shared_spaces` in `config.json`: `name`, `root`,
   `members`, and `tasks` (`data_source_url` plus the `properties`/
   `status_values` mapping — including `waiting` when one was inferred —
   and the `confirmed`/`unconfirmed_roles` marker), matching the exact
   shape in `schema.md`'s config example.

In non-interactive/Routine mode, skip prompting and leave `shared_spaces` as
whatever is already in the adopted `config.json` (do not invent spaces).

### 8. Query-gating test

Issue one single-source filtered+sorted `notion-query-data-sources` query
against a scratch data source (or the Inbox data source if no scratch DB is
available) — never a cross-data-source query, per
`notion-conventions.md`.

- **Succeeds:** record the query path as `structured` (the workspace is on
  Business + Notion AI or higher — a single-source query is ungated there).
- **Throws the plan-gate error** (the `400` signature in
  `query-plan-gating.md` — a single-source query throwing it means a
  Plus/Free tier): fall back to scoped `notion-search` + `notion-fetch`,
  record the query path as `fallback`, and surface the
  upgrade-vs-permanent-fallback decision to the user (don't silently pick
  one). A generic 400 from a malformed probe filter is a bug in the test, not
  a plan gate — fix the probe rather than recording `fallback`.

Store the observed path in the run report — it's advisory for other
query-dependent skills, not written into `config.json`'s schema.

### 9. Write config.json

Write `config.json` to the launch folder with every key defined in
`schema.md`: `user`, `notion_user_id`, `second_brain_root`, `home_page`,
`wiki` (`database_id`, `data_source_url`), `inbox`, `tasks_personal`,
`journal`, `archive` (each with `data_source_url`, except `tasks_personal`
when unresolved — see below), `shared_spaces` (array,
possibly empty), and `preferences` (`timezone`, `calendar_tool`). **Keep
`timezone` in the key set** — it must not be dropped. Use the exact key
names from `schema.md` — do not add, rename, or drop keys. `tasks_personal`
and every `shared_spaces[].tasks` now also carry `properties` +
`status_values` (plus `confirmed`/`unconfirmed_roles`) alongside
`data_source_url`, per §3b/§7 and the exact shape in
`schema.md`/`task-db-mapping.md` — except when §3b left `tasks_personal`
unresolved (2+ ambiguous candidates, non-interactive), in which case write
`tasks_personal` as `{"pending_selection": true, "candidates": [...]}` with
no `data_source_url`, mirroring how `unconfirmed_roles` marks a mapping
incomplete.

**Derive and persist `preferences.timezone`.** After `notion-get-users`
(§1), if the loaded/adopted config has no `preferences.timezone` set:
interactively, ask the user for their IANA timezone (e.g. `America/New_York`)
and write it. In Routine/non-interactive mode, keep whatever timezone the
adopted config already has — **never** fall back to the running session's
clock, which is UTC in cloud/Routine/Cowork sessions (`task-db-mapping.md`).
If there is no adopted config and no way to prompt, leave `timezone` unset
and report it as a pending manual step (§10) rather than guessing one.

Mirror the identical JSON into Home's fenced config block (§6, via
`insert_content`).

`config.json` is gitignored — never commit it, and never write personal
data (names, IDs, workspace names) anywhere else in this skill's own output
files.

### 10. Idempotency + Routine mode

Re-running `setup` on an already-populated folder/workspace must change
nothing that's already present and correct — it re-verifies each step,
finds everything present, and reports "all present": still additively
patching internal DBs per step 3's rules where properties are missing, and,
for task DBs, reading the stored mapping (§2) rather than re-discovering or
mutating anything (§3b, §7).

If invoked non-interactively (Routine mode, no way to prompt for the wiki
URL, the Waiting-option confirmation, or task-DB mapping disambiguation): do
not block waiting for input. Instead, produce a report listing exactly what
setup *would* create or still needs from the user (e.g. "wiki not yet turned
into a wiki — visit Home for instructions", "Waiting status option still
missing on Tasks", "task DB mapping has unconfirmed roles: due, priority —
pending pick"), and complete the rest of the run (databases, Home,
config.json, gating test) as far as it can go non-interactively.

## Smoke test

Smoke test (run against a live Notion workspace):
1. From an empty launch folder (no config.json), run setup.
2. Expect setup to: ping the connector (notion-get-users) OK; find or create the
   "Second Brain" root; create/adopt Inbox, Journal, Archive with the
   schema.md schemas; discover-or-scaffold the personal task DB per §3b;
   create/adopt Home with a fenced config block; write config.json with all
   keys populated.
3. Expect setup to run the gating test: a single-source filtered+sorted query
   against a scratch DB, and report path = "structured" or "fallback".
4. Expect setup to surface the manual UI steps (create 📚 Wiki → "Turn into
   wiki" → paste URL back; add "Waiting" status option — only if the personal
   DB was scaffolded), then verify each after the user confirms.
5. Re-run setup on the now-populated folder → it changes nothing (idempotent) and
   reports "all present".
Assertions: config.json exists with non-empty second_brain_root, inbox,
tasks_personal, journal, archive; Notion shows the four DBs with correct
properties (task DB unchanged from before the run — no DDL against it);
Home contains the config block; `tasks_personal`'s mapping is recorded with
`properties`/`status_values` and a `confirmed` marker — identity-mapped if
scaffolded, discovered if adopted; a re-run with no local `config.json`
reads the mapping from the Home block and skips re-disambiguation; a
non-interactive run records `unconfirmed_roles` for any ambiguous role and
reports it as a pending pick instead of guessing.

## Errors

- **Connector unavailable/unauthenticated** (`notion-get-users`, or any
  other Notion MCP call, fails): stop immediately, state the failure
  plainly, and point the user at claude.ai → Settings → Connectors → Notion
  to connect or reconnect it. Never fake success or continue as if the
  workspace were reachable.
- **Multiple root candidates:** stop and ask the user which `Second Brain`
  page to use. Never guess.
- **Property type conflict on an existing internal DB (Inbox/Journal/
  Archive):** never mutate it — report the conflicting database and
  property, skip that database, and continue with the rest of the run. Task
  DBs never receive DDL, so a "wrong type" on a task DB isn't a
  mutation-blocked case — see the next error.
- **Un-identifiable `title`/`status` on a task DB** (personal or shared):
  report and skip that DB — never mutate it, and never guess a role. Record
  it as unmapped rather than fabricating a mapping.
- **Rate limiting (`429`):** honor `Retry-After` and back off before
  retrying, per `notion-conventions.md`.
- **Large/async writes:** if a write returns an async task, poll
  `notion-get-async-task` rather than assuming completion.
