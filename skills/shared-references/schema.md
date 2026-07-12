# Notion Second Brain — Canonical Schema

Emoji and page/DB display names below are the **convention**; the real names and
IDs are recorded in `config.json` by `setup`. Skills address databases by the
`data_source_url` in config, never by display name (except during discovery).

## Structure (per person, under a private root)

```
🧠 Second Brain              ← root; named exactly "Second Brain" (emoji prefix ok)
├── ⚡ Inbox      [database]
├── ✅ Tasks      [database]
├── 📚 Wiki       [wiki]      ← created via "Turn into wiki" (UI only)
├── 📓 Journal    [database]
├── 🗑 Archive    [database]  ← discard target (no page-trash tool exists)
└── 🏠 Home       [page]      ← dashboard views + fenced config block
```

Shared areas (zero or more, registered in setup) each contain a `Tasks` database
with the identical Task schema below.

## Task database schema (personal + every shared, identical)

| Property | Type | Notes |
|---|---|---|
| Name | title | |
| Status | status | Notion defaults **Not started / In progress / Done**, plus **Waiting** added manually (see below) |
| Due | date | |
| Assignee | person | |
| Priority | select | |
| Project | select | |
| Source | url or rich_text | capture provenance |

**Status options:** MCP can only create the three default options; **Waiting** is
added by a one-time manual UI step (`setup` instructs and verifies it) into the
**In progress** group. Never attempt to add or rename status options via MCP.

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
expiry days). New wiki pages must parent to the wiki **page** URL, not the data
source.

## Adopt / patch rules (used by `setup`)

- Never re-create anything that exists. Match by display-name convention during
  discovery, then record the real ID/URL in config.
- For an existing database: verify each required property above exists with the
  right type; **add** missing properties (`notion-update-data-source` DDL);
  never drop or retype existing ones.
- If a required property exists with a conflicting type, do **not** mutate —
  report the conflict and stop for that database.

## `config.json` (written by `setup`, gitignored, launch-folder)

```json
{
  "user": "<name>",
  "notion_user_id": "…",
  "second_brain_root": "page_id",
  "home_page": "page_id",
  "wiki": { "database_id": "…", "data_source_url": "collection://…" },
  "inbox": { "data_source_url": "…" },
  "tasks_personal": { "data_source_url": "…" },
  "journal": { "data_source_url": "…" },
  "archive": { "data_source_url": "…" },
  "shared_spaces": [
    {
      "name": "<space name>",
      "root": "…",
      "tasks": { "data_source_url": "…" },
      "members": [
        { "name": "<person A>", "notion_user_id": "…" },
        { "name": "<person B>", "notion_user_id": "…" }
      ]
    }
  ],
  "preferences": { "timezone": "America/New_York", "calendar_tool": null }
}
```

`home_page` and `journal` extend the original design's example (needed by the
discovery fallback and by `today`, respectively).

## Discovery config-block format (fallback when no `config.json`)

When no `config.json` is found, skills `notion-search` for the root page named
exactly `Second Brain` (emoji prefix allowed), owned by / private to the current
user, containing a `Home` child page. `Home`'s body includes a fenced ```json
block with the **same keys as `config.json` above**. One `notion-fetch` of Home
yields every ID. On ambiguity or no match: **fail loudly** with setup
instructions — never guess.
