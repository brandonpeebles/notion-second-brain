# Notion Second Brain — Canonical Schema

Page/DB titles are **plain names** (no emoji in the title); the emoji shown below
is each object's **native Notion icon**, set separately (see Icons in
`notion-conventions.md`). Real names and IDs are recorded in `config.json` by
`setup`. Skills address databases by the `data_source_url` in config, never by
display name (except during discovery).

## Structure (per person, under a private root)

Icons shown are native icons; titles carry no emoji.

```
🧠 Second Brain              ← root; title exactly "Second Brain" (leading emoji tolerated on adopt)
├── ⚡ Inbox      [database]
├── ✅ Tasks      [database]
├── 📚 Wiki       [wiki]      ← created via "Turn into wiki" (UI only)
├── 📓 Journal    [database]
├── 🗑 Archive    [database]  ← discard target (no page-trash tool exists)
├── 🏠 Home       [page]      ← human dashboard (quick links only)
└── 🤖 AGENTS     [page]      ← AGENTS.md-style control file: config block + Notion AI instructions
```

Shared areas (zero or more, registered in setup) each contain a task database
whose schema is whatever already exists there; setup records a mapping (see
`task-db-mapping.md`) rather than forcing it to match the shape below.

## Task database schema — scaffold default (used only when setup CREATES a task DB)

Adopted task DBs keep their real property names and status values; setup
records a mapping for them (see `task-db-mapping.md`) instead of applying the
shape below. The table below is only the shape `setup` scaffolds when no
existing task DB is found.

| Property | Type | Notes |
|---|---|---|
| Name | title | |
| Status | status | Notion defaults **Not started / In progress / Done**, plus **Waiting** added manually (see below) |
| Due | date | |
| Assignee | person | |
| Priority | select | |
| Project | select | |
| Source | url or rich_text | capture provenance |

**Status options (scaffolded DBs only):** MCP can only create the three default
options; **Waiting** is added by a one-time manual UI step (`setup` instructs
and verifies it) into the **In progress** group. Never attempt to add or
rename status options via MCP — and never add a status option to an adopted
DB.

## Inbox database schema

| Property | Type | Notes |
|---|---|---|
| Name | title | |
| Type | select | Task / Note / Idea / Reference — the capturer's guess |
| Source | url or rich_text | where it came from |
| Captured | created_time | |
| Triage | select | New / Kept / Promoted — set by `triage` |

## Journal database schema (minimal; extensible in v0.3 `fold`)

| Property | Type | Notes |
|---|---|---|
| Name | title | e.g. the date or a short label |
| Date | date | |
| Type | select | Daily / Digest |

`today` upserts the single `Type = Daily` row per date, matched by **Date +
`Type` = `Daily`** (not Date alone). Items promoted into the Journal by
`triage` are separate dated rows with `Type` left **unset**, so they never
collide with `today`'s Type-scoped upsert. `fold` will later add Digest rows
and a dedicated note `Type` (v0.3) and may extend this schema.

## Archive database schema

| Property | Type | Notes |
|---|---|---|
| Name | title | preserved from the archived row |
| Archived | created_time | |
| Origin | rich_text | where the row came from (e.g. "Inbox") |

## Wiki

The wiki is a Notion wiki (databases under the hood). Agent-facing metadata =
title, timestamps, Owner, Verification, hierarchy, content. Custom properties
(e.g. Tags) are **human-only** — MCP cannot set them on wiki pages; only
Verification is settable (`notion-update-page` `update_verification`, optional
expiry days). `wiki.database_id` identifies the wiki **page** and is the
**parent target** for creating new wiki pages; `wiki.data_source_url` is used
only for structured queries against the wiki. New wiki pages must parent to
`wiki.database_id` (the wiki page), never to `wiki.data_source_url`.

## Adopt / patch rules (used by `setup`)

Never re-create anything that exists. Match by display-name convention during
discovery (tolerating a legacy leading emoji in the title), then record the real
ID/URL in config. On adopt, also **repair the icon/title**: if the object's
title carries a leading emoji and/or it has no native icon, set the native icon
and strip the emoji from the title (see Icons in `notion-conventions.md`). Two
modes apply for property patching, depending on the database:

**Internal DBs (Inbox / Journal / Archive):**

- For an existing database: verify each required property above exists with the
  right type; **add** missing properties (`notion-update-data-source` DDL);
  never drop or retype existing ones.
- If a required property exists with a conflicting type, do **not** mutate —
  report the conflict and stop for that database.

**Task DBs (personal + shared): map, never mutate.** See
`task-db-mapping.md` for the full contract. Setup discovers the DB's actual
schema and records it — via `properties` + `status_values` — rather than
patching the DB to match a canonical shape:

- Never add, rename, retype, or drop a property or status option on a task DB.
- The personal DB only, when none exists, may be **scaffolded** (the shape
  above) and given an identity mapping.
- If `title` or `status` can't be identified, fail loudly and stop for that
  DB — per `task-db-mapping.md`'s fail-loudly rule — rather than guessing or
  mutating.

## `config.json` (written by `setup`, gitignored, launch-folder)

```json
{
  "user": "<name>",
  "notion_user_id": "…",
  "second_brain_root": "page_id",
  "home_page": "page_id",
  "agents_page": "page_id",
  "wiki": { "database_id": "…", "data_source_url": "collection://…" },
  "inbox": { "data_source_url": "…" },
  "tasks_personal": {
    "data_source_url": "collection://…",
    "confirmed": true,
    "properties": { "title":"Name","status":"Status","due":"Due","assignee":"Assignee","priority":"Priority","project":"Project","source":"Source" },
    "status_values": { "open_default":"Not started", "done":["Done","Archived"] }
  },
  "journal": { "data_source_url": "…" },
  "archive": { "data_source_url": "…" },
  "shared_spaces": [
    {
      "name": "<space name>",
      "root": "…",
      "members": [
        { "name": "<person A>", "notion_user_id": "…" },
        { "name": "<person B>", "notion_user_id": "…" }
      ],
      "tasks": {
        "data_source_url": "collection://…",
        "confirmed": true,
        "properties": { "title":"Task","status":"Status","due":"Deadline","scheduled":"Do date","assignee":"Assignee","priority":"Priority" },
        "status_values": { "open_default":"Not started","done":["Will not do","Done"],"waiting":"Awaiting response" }
      }
    }
  ],
  "preferences": { "timezone": "America/New_York", "calendar_tool": null }
}
```

If `setup` ran non-interactively and found 2+ ambiguous personal task-DB
candidates, `tasks_personal` instead takes the unresolved-pick shape
`{"pending_selection": true, "candidates": [...]}` (no `data_source_url`) —
see `task-db-mapping.md`.

`home_page` is the human dashboard (quick links). `agents_page` is the
AGENTS.md-style control file that carries the fenced config block and doubles as
the workspace's Notion AI instructions page — it is the discovery anchor (see
below). `journal` extends the original design's example (needed by `today`).
Each task DB object
(`tasks_personal` and every `shared_spaces[].tasks`) also carries `properties`
(role → real property name, roles omitted when absent) and `status_values`
(role → real status value), plus the `confirmed` marker — see
`task-db-mapping.md` for the full contract.

## Discovery config-block format (fallback when no `config.json`)

When no `config.json` is found, skills `notion-search` for the root page named
exactly `Second Brain` (emoji prefix allowed), owned by / private to the current
user, containing an `AGENTS` child page (title `AGENTS`, a leading emoji
tolerated). `AGENTS`'s body includes a fenced ```json block with the **same keys
as `config.json` above**, including each task DB's
`properties`/`status_values`/`confirmed` mapping. One `notion-fetch` of AGENTS
yields every ID. On ambiguity or no match: **fail loudly** with setup
instructions — never guess.
