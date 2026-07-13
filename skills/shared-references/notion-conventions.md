# Notion MCP Conventions & Live-Learned Quirks

Learned via live tests 2026-07-11; encode here so they're never re-derived.

## Querying
- **Single data source per query, always.** `notion-query-data-sources` takes one
  data source. To span personal + shared task DBs, query each in turn and merge in
  the skill. Never issue a cross-data-source query.
- **Dual path.** Primary: structured `notion-query-data-sources` (filter + sort).
  Fallback: scoped `notion-search` + `notion-fetch` of known DBs. The plan gate
  surfaces as a **thrown `400` `APIResponseError`**, not an inspectable "upgrade"
  result — catch the error, switch to the fallback, note the degradation in the
  output, and surface the "upgrade vs. permanent fallback" decision. See
  `query-plan-gating.md` for the exact error signature, the per-tier gate map,
  and the full decision tree. Every query-dependent skill smoke-tests both paths.
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

## Icons
- Set a **native Notion icon**; never embed an emoji in the title. An emoji in
  the title *and* the object's own icon slot renders as two icons side by side.
- `notion-create-pages` and `notion-update-page` both take an `icon` field (an
  emoji, a `:custom_emoji:` name, or an external image URL). Create pages with a
  plain title + `icon`.
- `notion-create-database` has **no** `icon` param. Create the DB with a plain
  title, then set its icon with a follow-up `notion-update-page` against the
  **database page id**. To change only the icon (no property edit), call
  `command: "update_properties"` with `properties: {}` and the `icon` field — a
  bare icon call errors with *"update_properties requires a properties
  parameter."* (Verified live: setting a database's icon this way succeeds.)
- **Repair on adopt:** if an existing object has an emoji in its title (and/or no
  native icon), set the native icon and strip the leading emoji from the title.
  Idempotent — a plain title + icon is left unchanged.

## AI instructions (UI-only)
- Designating a page as the workspace's **Notion AI Instructions** page
  (page `···` → `Use with AI` → `Use as AI Instruction`, or Settings → Notion AI
  → Add Instructions) is **UI-only** — no MCP/API tool sets it. Only **one**
  instructions page is active at a time (setting a new one replaces the current).
  Not verifiable via MCP; treat like "Turn into wiki" and the Waiting option —
  guide the user, accept their confirmation, report as pending in Routine mode.

## Status properties
- DDL can `ADD COLUMN "X" STATUS` but **only with Notion defaults** (Not started /
  In progress / Done). Custom options are rejected on ADD and ALTER; the three
  groups are immutable via API. **Waiting** is added by a one-time manual UI step.

## Trash / discard
- There is **no page-level trash/delete tool.** Discard a row by moving it into the
  `Archive` DB with `notion-move-pages` (or a property change). Data *sources*
  can be trashed via `notion-update-data-source {in_trash}` — not rows.

## Referencing pages inline
Two renderings, by audience — the source of truth for the Notion syntax is the MCP
resource `notion://docs/enhanced-markdown-spec` (read it; don't guess mention syntax).
- **In a Notion page/DB body** — use native mentions: `<mention-page url="URL"/>`
  renders the page's live title + icon and **creates a backlink on the referenced
  page**; `<mention-date start="YYYY-MM-DD"/>` renders a native date pill. The
  self-closing form shows the UI-resolved title, so you don't have to pass one.
- **In plain-text / chat (session) output** — use a `[title](url)` markdown link
  (the same "page/row title + Notion link" citation pattern `query` already uses).
  **Never emit `<mention-*>` tags to chat** — they render as raw XML outside Notion.
- **Getting the URL:** every queried row carries a page id/URL on both the
  structured (`notion-query-data-sources`) and fallback (`notion-search` /
  `notion-fetch`) paths. Prefer the row's own URL; else construct
  `https://www.notion.so/<id>` from the page id.
- **Idempotency caveat:** a re-fetched body serializes mentions/date pills back in
  *resolved* form, so any in-place `update_content` must match against the
  **freshly-fetched** body text, never the string you originally wrote (this is
  already how a fetch-then-replace sequence like `today` §4→§5 is ordered).

## Views
- `notion-create-view` adds a saved view to an existing database (`database_id`
  + `data_source_id`) or an inline linked-database view on a page
  (`parent_page_id` + `data_source_id`). `notion-update-view` edits an
  existing view's name/filter/sort by `view_id`. Both take a small DSL in
  `configure`, e.g. `FILTER "Triage" = "New"` — see the
  `notion://docs/view-dsl-spec` MCP resource for the full syntax.
- **Idempotent creation:** before creating a view, `notion-fetch` the database
  and check its `<views>` list for one already named the same — never create a
  duplicate.
- **Linking to a view from elsewhere:** a view has its own `view://…` URI
  (returned by `notion-create-view`/shown in a database's `<views>` list) and
  a human-clickable form — the database's page URL with `?v=<view-id-no-dashes>`
  appended. Use the `view://` form when passing the view to another MCP tool
  (e.g. `notion-update-view`).
- **`?v=` links do not survive as page-content mentions.** A markdown link
  `[text](url?v=…)` written into a page's body is resolved to a live page
  mention on write, and the mention **drops the `?v=` query string** — a
  re-fetch shows the bare page URL (verified live: the param is gone
  immediately after the write that set it, not just on a later re-fetch). Do
  **not** rely on this to deep-link a Home-page quick link to a specific view.
  To make a view the one a page-mention click lands on, reorder the
  database's views in the Notion UI so the desired view is leftmost/default
  (not exposed via MCP) — otherwise just link to the database's page, which
  opens whichever view is currently first.

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
