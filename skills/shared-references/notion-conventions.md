# Notion MCP Conventions & Live-Learned Quirks

Learned via live tests 2026-07-11; encode here so they're never re-derived.

## Querying
- **Single data source per query, always.** `notion-query-data-sources` takes one
  data source. To span personal + shared task DBs, query each in turn and merge in
  the skill. Never issue a cross-data-source query.
- **Dual path.** Primary: structured `notion-query-data-sources` (filter + sort).
  Fallback: scoped `notion-search` + `notion-fetch` of known DBs. If the structured
  tool returns an upgrade/plan-gating prompt, switch to the fallback, note the
  degradation in the output, and surface the "Business upgrade vs. permanent
  fallback" decision. Every query-dependent skill smoke-tests both paths.
- `notion-search` without Notion AI is workspace-scoped only — acceptable; note it.

## Wikis
- Wikis are databases under the hood and are structured-queryable
  (`collection://…`), exposing Page / Owner / Tags / Verification / Last edited time.
- **Cannot be created via API/MCP** — "Turn into wiki" is UI-only.
- **Cannot set custom property values** (e.g. Tags) on wiki pages via MCP. Only
  **Verification** is settable: `notion-update-page` has an `update_verification`
  command with optional expiry days. Agent-written metadata therefore lives in real
  databases, not wiki properties.
- Create pages in a wiki by parenting to the wiki **page** URL, not the data source.

## Status properties
- DDL can `ADD COLUMN "X" STATUS` but **only with Notion defaults** (Not started /
  In progress / Done). Custom options are rejected on ADD and ALTER; the three
  groups are immutable via API. **Waiting** is added by a one-time manual UI step.

## Trash / discard
- There is **no page-level trash/delete tool.** Discard a row by moving it into the
  `🗑 Archive` DB with `notion-move-pages` (or a property change). Data *sources*
  can be trashed via `notion-update-data-source {in_trash}` — not rows.

## Writes & concurrency
- Prefer `update_content` / `insert_content` and property updates over whole-page
  replacement to avoid clobbering concurrent human edits.
- Large writes may return an async task — poll `notion-get-async-task` when so.

## Data-source model & rate limits
- Notion's Sept-2025 data-source model: a database has one or more data sources;
  skills address them by `data_source_url` (`collection://…`) from config.
- ~3 requests/second per connection + a secondary per-workspace limit. On `429`,
  honor `Retry-After` and back off. Batch reads; prefer one structured query over
  many fetches.

## Connector identity
- Hosted MCP OAuth is per-user per-workspace; the connector acts as the logged-in
  person, so private-page permissions are enforced automatically. Guests are
  blocked from MCP — both people need member seats.
