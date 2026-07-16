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
├── 📨 Raw        [database]  ← capture store; "Inbox" is a saved view on it (Triage = New)
├── ✅ Tasks      [database]
├── 📚 Wiki       [wiki]      ← created via "Turn into wiki" (UI only)
├── 📓 Journal    [database]
├── 🗑 Archive    [database]  ← discard target (no page-trash tool exists)
├── 🏠 Home       [page]      ← human dashboard (DB links + everyday-workflow guide)
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

## Raw database schema (the Inbox store)

The plugin's single capture store is a database titled **Raw** — it holds every
captured item, unprocessed and processed alike; processing changes a row's
`Triage`, it never deletes the row. **"Inbox" is not a separate database** — it's
a saved **view** on Raw, filtered to `Triage =` the `new` value, that `capture`
feeds and `triage` sweeps (see "The Inbox view" below). Skills still address this
store via the single `inbox.data_source_url` config key regardless of what the
underlying database happens to be titled.

| Property | Type | Notes |
|---|---|---|
| Name | title | |
| Type | select | Task / Note / Idea / Reference — the capturer's guess. `capture` only ever picks among these four; extra hand-added options (e.g. Article, Meeting notes) are left alone |
| Source | url or rich_text | where it came from |
| Captured | created_time | |
| Triage | select | Values resolved via `inbox.triage_values` (see below); default `New` / `Processed` — set by `triage` |

Additive human-added properties (e.g. a `Tags` multi-select) are common and left
alone — the plugin never creates, requires, or writes them.

### The "Inbox" view

`setup` creates (when scaffolding) or adopts (when discovering) a saved view
named exactly **Inbox** on Raw, filtered to `Triage =` the `new` value resolved
from `inbox.triage_values`, via `notion-create-view`/`notion-update-view` (see
`notion-conventions.md`). This view has no effect on `capture`/`triage` logic —
they already query/filter `inbox.data_source_url` directly and never depend on
the view existing. Its URL is recorded as `inbox.view_url` (optional) purely so
a re-run can verify/update it idempotently without re-searching — **not** for
Home-page linking: a `?v=` view link does not survive as page content (see
`notion-conventions.md`), so the Home page's "Inbox" quick link always points
to Raw's own page, same as "Raw" does. Making Notion open straight to the
Inbox view on click requires reordering Raw's views in the UI so Inbox is
leftmost/default — a manual step outside MCP's reach, not something `setup`
can do or verify.

### Inbox `Triage` values (config-resolved)

The Inbox `Triage` **property name** is canonical and fixed (always `Triage`) —
its name is never mapped. Its **enum values**, however, are resolved through an
optional `inbox.triage_values` map in `config.json`, mirroring how task DBs
resolve `status_values` (`task-db-mapping.md`). Roles:

- `new` — the "unprocessed" value `capture` writes and `triage` filters the
  queue on.
- `processed` — the value `triage` writes when it promotes a row to a Task,
  Wiki page, or Journal entry.
- `kept` — the value `triage`'s "Keep in Inbox" outcome writes. **Optional:**
  when `kept` is absent from the map, the Keep outcome writes nothing and leaves
  the row at the `new` value, so it resurfaces in the next sweep. This is how the
  default two-value `New` / `Processed` scheme collapses the Keep outcome.

**Default when `inbox.triage_values` is absent:**
`{ "new": "New", "processed": "Processed" }` — the two-value scheme, matching a
freshly scaffolded Raw database. A workspace still running the plugin's original
three-value scheme sets `"triage_values": { "new": "New", "processed":
"Promoted", "kept": "Kept" }` explicitly.

Skills resolve these roles through this map and **never** hard-code a `Triage`
value. This file is the single home for the role contract and the default —
skills reference it rather than restating the default.

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
| Origin | rich_text | where the row came from (e.g. "Raw") |

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

**Internal DBs (Raw / Journal / Archive):**

- For an existing database: verify each required property above exists with the
  right type; **add** missing properties (`notion-update-data-source` DDL);
  never drop or retype existing ones.
- If a required property exists with a conflicting type, do **not** mutate —
  report the conflict and stop for that database.
- **Raw discovery convention:** search for a child titled exactly `Raw` first
  (leading emoji tolerated). If not found, fall back to a child titled `Inbox`
  — a pre-consolidation workspace — and adopt it as-is under its own title,
  never force-renamed to `Raw`. Only when **neither** is found does setup
  scaffold a fresh `Raw` database. Either way, `inbox.triage_values` is decided
  by inspecting the DB's actual `Triage` select options (see setup §3), never
  by which title was matched.

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

## `config.json` (written by `setup`, committed in the private second-brain repo)

```json
{
  "user": "<name>",
  "notion_user_id": "…",
  "second_brain_root": "page_id",
  "home_page": "page_id",
  "agents_page": "page_id",
  "wiki": { "database_id": "…", "data_source_url": "collection://…" },
  "inbox": { "data_source_url": "…", "triage_values": { "new": "New", "processed": "Processed" }, "view_url": "view://…" },
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
  "preferences": { "timezone": "America/New_York", "calendar_tool": null, "email_tool": null },
  "email": {
    "scan_query": null,
    "unread_only": false,
    "important_only": false,
    "window_days_cap": 3,
    "auto_extract": false,
    "awaiting_reply_days": 5,
    "awaiting_lookback_days": 30,
    "wiki_match": false,
    "watch":  { "topics": [], "senders": [] },
    "ignore": { "topics": [], "senders": [] }
  }
}
```

If `setup` ran non-interactively and found 2+ ambiguous personal task-DB
candidates, `tasks_personal` instead takes the unresolved-pick shape
`{"pending_selection": true, "candidates": [...]}` (no `data_source_url`) —
see `task-db-mapping.md`.

`home_page` is the human dashboard (DB links plus a short everyday-workflow
guide). `agents_page` is the
AGENTS.md-style control file that carries the fenced config block and doubles as
the workspace's Notion AI instructions page — it is the discovery anchor (see
below). `journal` extends the original design's example (needed by `today`).
Each task DB object
(`tasks_personal` and every `shared_spaces[].tasks`) also carries `properties`
(role → real property name, roles omitted when absent) and `status_values`
(role → real status value), plus the `confirmed` marker — see
`task-db-mapping.md` for the full contract.

`inbox.triage_values` is **optional** — a role→value map for the Inbox `Triage`
select (see "Inbox `Triage` values" above). Omit it to use the default two-value
`New` / `Processed` scheme; set it explicitly to
`{ "new": "New", "processed": "Promoted", "kept": "Kept" }` to keep the plugin's
original three-value scheme.

`inbox.view_url` is **optional** — the saved "Inbox" view on Raw (see "The
Inbox view" above), used only to render the Home page's Inbox quick link. Its
absence never blocks `capture`/`triage`, which address Raw directly via
`inbox.data_source_url`.

`preferences.calendar_tool` is **optional** — `null` (the default) auto-detects a
session calendar connector (Google Calendar, `mcp__claude_ai_Google_Calendar__*`);
`"google_calendar"` pins it. `today` writes the detected value back on first
discovery — durable: `config.json` **and** the AGENTS config block; ephemeral: the
AGENTS block only (see `durability-modes.md`) — so later runs skip re-discovery.
When no calendar tool is set and none is available, the Calendar section is
omitted, never an error (see `today`'s §3).

`preferences.email_tool` is **optional** — `null` (the default) auto-detects the
Gmail connector (`mcp__claude_ai_Gmail__*`) in the session; `"gmail"` pins it. It
mirrors `preferences.calendar_tool`. When no email tool is set and none is
available, email scanning is omitted, never an error (see `email.md`).

The `email` block is **optional** — every key defaults per the example above when
the block (or any key) is absent, so existing workspaces keep working with **no
re-`setup`**. `scan_query` (non-null) replaces the base scan query wholesale;
`unread_only`/`important_only` are composable toggles; `window_days_cap` bounds
the scan window (default 3 days); `auto_extract` gates unattended high-confidence
writes to Raw — **off by default**, so qualifying confirmations are *surfaced* for
on-demand extraction rather than written (set `true` to write them unattended);
`awaiting_reply_days` (default 5) is the staleness threshold for the "Waiting — no
response yet" sweep and `awaiting_lookback_days` (default 30) bounds how far back that
sweep scans Sent mail; `wiki_match` opts into a per-candidate `notion-search`; `watch`/
`ignore` are topic/sender lists. The full behavioral contract for every key lives
in `email.md` — this file only fixes the shape. **`last_scan_ts` is deliberately
NOT in this block** — it is runtime state on the AGENTS page (see "Agent state
block" below and `email.md`), so daily scanning never churns `config.json`.

## Discovery config-block format (fallback when no `config.json`)

When no `config.json` is found, skills `notion-search` for the root page named
exactly `Second Brain` (emoji prefix allowed), owned by / private to the current
user, containing an `AGENTS` child page (title `AGENTS`, a leading emoji
tolerated). `AGENTS`'s body includes a fenced ```json block with the **same keys
as `config.json` above**, including each task DB's
`properties`/`status_values`/`confirmed` mapping. One `notion-fetch` of AGENTS
yields every ID. On ambiguity or no match: **fail loudly** with setup
instructions — never guess.

## AGENTS-page *Agent state* block (runtime state, not config)

Beyond the fenced ```json **config** block (above), the AGENTS page also carries
a **separate** fenced ```json block under an `## Agent state` heading, holding
runtime state that must **not** live in `config.json` (so daily writes don't churn
git) and must **not** live in the config block (so it is exempt from the
config↔config-block sync invariant — see `durability-modes.md`). Shape:

```json
{ "email": { "last_scan_ts": "2026-07-13T18:30:00Z" } }
```

`last_scan_ts` is an absolute UTC instant (ISO-8601 with explicit offset). The
block is created **lazily** on the first email scan (not by `setup`), read via
`notion-fetch` and rewritten in place via `update_content`, and works identically
in `durable` and `ephemeral` modes. The full state contract (advance-on-success,
timezone rule, gap guard) lives in `email.md`.
