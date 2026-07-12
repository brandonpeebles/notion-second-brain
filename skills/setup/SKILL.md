---
name: setup
description: Use when setting up or repairing the Notion second brain — discovers, scaffolds, and adopts the workspace structure, writes config.json, verifies the Notion connector, runs the query-gating test, and registers shared spaces. Idempotent and additive; safe to re-run to add spaces.
---

# setup

Foundational skill for the Notion second brain. Discovers or scaffolds the
workspace structure, adopts what already exists, writes `config.json`,
verifies the Notion connector, runs the query-gating test, and walks the user
through two UI-only manual steps. Its successful run produces the live
workspace every other skill (`capture`, `today`, `triage`, `query`) operates
on. Idempotent and additive: re-running never destroys anything, and can be
used to add more shared spaces later.

Consult `../shared-references/schema.md` for the canonical DB schemas,
property names/types, the `config.json` key set, adopt/patch rules, and the
discovery config-block format. Consult `../shared-references/notion-conventions.md`
for MCP quirks (wiki creation, status defaults, no-trash, single-source
queries, dual-path fallback, rate limits, async writes, connector identity).
Use the config keys and property names exactly as `schema.md` defines them —
do not paraphrase or rename.

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
the current user, per the discovery convention in `schema.md`.

- **Exactly one match:** adopt it as the root.
- **No match:** offer to create `🧠 Second Brain` (`notion-create-pages`),
  private to the current user, and use it as the root.
- **Multiple matches:** stop and ask the user which one to use. Never guess.

### 3. Scaffold or adopt the four databases

Under the root, scaffold or adopt Inbox, Tasks, Journal, and Archive using
the schemas in `schema.md` and its adopt/patch rules:

- For each database, `notion-search`/`notion-fetch` under the root for a
  child matching the display-name convention (`⚡ Inbox`, `✅ Tasks`,
  `📓 Journal`, `🗑 Archive`).
- **Missing:** create it with `notion-create-database` using the exact
  properties and types from `schema.md`.
- **Existing:** verify every required property exists with the right type;
  **add** any missing properties with `notion-update-data-source` (DDL).
  Never drop or retype an existing property.
- **Type conflict** (a required property exists with the wrong type): do
  **not** mutate it. Report the conflict for that database and skip it —
  continue with the remaining databases.

Record each database's `data_source_url` for `config.json`.

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
record `wiki.database_id` and `wiki.data_source_url` (`collection://…`) for
`config.json`. Remind the user (once, not per-run) that new wiki pages must
be parented to the wiki **page** URL, not its data source, and that custom
properties like Tags are human-only — MCP can only set Verification.

If a wiki is already found, adopt it directly (no manual step needed) and
skip to step 5.

In non-interactive/Routine mode (see §10), do not prompt for this step —
report it as a pending manual step instead.

### 5. Waiting status option (guided, UI-only)

After Tasks (personal, from step 3) is created or adopted, check whether its
Status property already has a **Waiting** option by fetching the Tasks data
source schema.

If it's missing, instruct the user to do a one-time manual UI step: open the
Tasks database in Notion, edit the Status property, and add a **Waiting**
option to the **In progress** group. MCP cannot add or rename status
options — Notion only allows creating the three defaults (Not started / In
progress / Done) via DDL (`notion-conventions.md`).

After the user confirms, re-fetch the Tasks data source schema and verify
**Waiting** is now present. Do not assume success from the user's word alone
— verify.

In non-interactive/Routine mode, do not prompt; report this as a pending
manual step instead.

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

1. Locate its `Tasks` database (search under the space's root, or ask the
   user for it).
2. Adopt/patch it to the identical Task schema from `schema.md`, using the
   same adopt/patch rules as step 3 (add missing properties, never mutate
   conflicting types — report and skip on conflict).
3. Populate `members` by resolving each named member via `notion-get-users`.
4. Append the space to `shared_spaces` in `config.json` (name, root, tasks
   data source, members).

In non-interactive/Routine mode, skip prompting and leave `shared_spaces` as
whatever is already in the adopted `config.json` (do not invent spaces).

### 8. Query-gating test

Issue one single-source filtered+sorted `notion-query-data-sources` query
against a scratch data source (or the Inbox data source if no scratch DB is
available) — never a cross-data-source query, per
`notion-conventions.md`.

- **Succeeds:** record the query path as `structured`.
- **Returns an upgrade/plan-gating prompt:** fall back to scoped
  `notion-search` + `notion-fetch`, record the query path as `fallback`, and
  surface the Business-upgrade-vs-permanent-fallback decision to the user
  (don't silently pick one).

Store the observed path in the run report — it's advisory for other
query-dependent skills, not written into `config.json`'s schema.

### 9. Write config.json

Write `config.json` to the launch folder with every key defined in
`schema.md`: `user`, `notion_user_id`, `second_brain_root`, `home_page`,
`wiki` (`database_id`, `data_source_url`), `inbox`, `tasks_personal`,
`journal`, `archive` (each with `data_source_url`), `shared_spaces` (array,
possibly empty), and `preferences` (`timezone`, `calendar_tool`). Use the
exact key names from `schema.md` — do not add, rename, or drop keys.

Mirror the identical JSON into Home's fenced config block (§6, via
`insert_content`).

`config.json` is gitignored — never commit it, and never write personal
data (names, IDs, workspace names) anywhere else in this skill's own output
files.

### 10. Idempotency + Routine mode

Re-running `setup` on an already-populated folder/workspace must change
nothing that's already present and correct — it re-verifies each step,
finds everything present, and reports "all present" (still applying any
missing properties per the adopt/patch rules in steps 3 and 7, since those
are additive and safe).

If invoked non-interactively (Routine mode, no way to prompt for the wiki
URL or the Waiting-option confirmation): do not block waiting for input.
Instead, produce a report listing exactly what setup *would* create or
still needs from the user (e.g. "wiki not yet turned into a wiki — visit
Home for instructions", "Waiting status option still missing on Tasks"),
and complete the rest of the run (databases, Home, config.json,
gating test) as far as it can go non-interactively.

## Smoke test

Smoke test (run against a live Notion workspace):
1. From an empty launch folder (no config.json), run setup.
2. Expect setup to: ping the connector (notion-get-users) OK; find or create the
   "Second Brain" root; create/adopt Inbox, Tasks, Journal, Archive with the
   schema.md schemas; create/adopt Home with a fenced config block; write
   config.json with all keys populated.
3. Expect setup to run the gating test: a single-source filtered+sorted query
   against a scratch DB, and report path = "structured" or "fallback".
4. Expect setup to surface the two manual UI steps (create 📚 Wiki → "Turn into
   wiki" → paste URL back; add "Waiting" status option), then verify each after
   the user confirms.
5. Re-run setup on the now-populated folder → it changes nothing (idempotent) and
   reports "all present".
Assertions: config.json exists with non-empty second_brain_root, inbox,
tasks_personal, journal, archive; Notion shows the four DBs with correct
properties; Home contains the config block.

## Errors

- **Connector unavailable/unauthenticated** (`notion-get-users`, or any
  other Notion MCP call, fails): stop immediately, state the failure
  plainly, and point the user at claude.ai → Settings → Connectors → Notion
  to connect or reconnect it. Never fake success or continue as if the
  workspace were reachable.
- **Multiple root candidates:** stop and ask the user which `Second Brain`
  page to use. Never guess.
- **Property type conflict on an existing database:** never mutate it —
  report the conflicting database and property, skip that database, and
  continue with the rest of the run.
- **Rate limiting (`429`):** honor `Retry-After` and back off before
  retrying, per `notion-conventions.md`.
- **Large/async writes:** if a write returns an async task, poll
  `notion-get-async-task` rather than assuming completion.
