---
name: query
description: Use when the user asks a question that should be answered from the second brain — runs structured queries and scoped semantic search across the wiki, tasks, and journal, returns a cited answer, and can optionally file the answer back.
---

# query

Answer a question **from** the second brain. Every question is routed to one
of two paths — structured filters over a known database, or scoped semantic
search over page/row content — and the final answer is grounded in whatever
was actually fetched, with citations. Optionally file the answer back to the
wiki or the Journal when asked.

Consult `../shared-references/schema.md` for the Wiki, Task, Journal, and
Inbox schemas (property names/types) and the `config.json` key set. Consult
`../shared-references/notion-conventions.md` for MCP quirks (single-source
queries, the dual-path rule, the wiki parent-URL quirk, rate limits, async
writes). Use the config keys and property names exactly as `schema.md`
defines them — do not paraphrase or rename.

## Behavior

### 1. Load config or discover

Same pattern as `capture`, `today`, and `triage`: look for `config.json` in
the launch folder first. If present and valid, read
`wiki.data_source_url`, `wiki.database_id` (the wiki **page** — needed only
for file-back, §6), `tasks_personal.data_source_url`,
`shared_spaces[].tasks` (data source URLs), `journal.data_source_url`,
`inbox.data_source_url`, `preferences.timezone`, and `notion_user_id` from
it.

If no `config.json` is found, fall back to discovery: `notion-search` for
the root page named exactly `Second Brain` (emoji prefix allowed), then
`notion-fetch` its `Home` child page and read the fenced ```json config
block for the same keys, per the discovery convention in `schema.md`.

If neither resolves: **fail loudly**. State plainly that the second brain
isn't set up yet, point the user at the `setup` skill, and **stop**. Never
guess at IDs, never invent a target, never assume a filesystem exists beyond
this one optional config-file check.

### 2. Pick the path per question

Every question gets routed before any tool call:

- **Structured** — the question maps onto known properties of a database:
  Tasks (`Status`, `Due`, `Assignee`, `Priority`, `Project`), Inbox (`Type`,
  `Triage`, `Captured`), Journal (`Date`, `Type`), or the Wiki's structured
  surface (`Page`, `Owner`, `Tags`, `Verification`, `Last edited time` — the
  Wiki is queryable as a data source per `notion-conventions.md`, but its
  **body content** is not). Examples: "what tasks are due this week", "what's
  overdue and assigned to me", "which wiki pages haven't been verified in 90
  days", "what's still New in the inbox".
- **Semantic / fallback** — the question is open-ended and depends on page or
  row **content**, not a filterable property: "what did we decide about the
  venue?", "what did the journal say about last week's trip?", "summarize the
  wiki page on X". Use scoped `notion-search` + `notion-fetch`.

Default to structured whenever the question maps onto DB properties; use the
semantic path for content questions. A single question may need both (e.g.
"what tasks are due this week for the trip?" — structured `Due` filter, plus
a semantic lookup to confirm which tasks are "for the trip" if that's not a
`Project` value) — run both and merge the grounding for one answer. When in
doubt about which path fits, prefer structured first (it's cheaper and more
precise) and fall through to semantic if it doesn't answer the question.

### 3. Structured path — single source per DB, merge in the skill layer

Issue one **single-source** `notion-query-data-sources` call per database
that's relevant to the question. Never issue a cross-data-source query —
each call targets exactly one `data_source_url`, per
`notion-conventions.md`.

- **Tasks spanning multiple DBs** (e.g. "what's due this week"): query
  `tasks_personal.data_source_url`, and separately each entry in
  `shared_spaces[]` (using that entry's `.tasks.data_source_url`), then merge
  all results in this skill after every call returns. Unless the question
  narrows to "my" tasks (in which case filter `Assignee` against
  `notion_user_id` after merging), return matches from every DB the question
  touches — a shared space's Tasks DB is visible to both members by design,
  so a query skill doesn't need `today`'s "assignee = me" narrowing unless
  asked for it.
- **Single-DB questions** (Inbox, Journal, Wiki): one single-source call
  against that data source's `data_source_url`.
- Translate the question into filters/sorts on real properties — e.g. "due
  this week" → `Due` between today and +7 days (resolved against
  `preferences.timezone`) AND `Status != Done`, sorted by `Due` ascending.
  Never invent a property that isn't in `schema.md`.

### 4. Semantic / fallback path — scoped search + fetch

Scope `notion-search` to the relevant area implied by the question (wiki
content, journal entries, inbox notes) rather than an unscoped workspace
search, then `notion-fetch` the top hits to read their actual content before
answering. Note that `notion-search` without Notion AI is workspace-scoped
only (per `notion-conventions.md`) — acceptable, but if the result set looks
too broad or misses an obviously-relevant page, say so rather than presenting
it as exhaustive.

**Degrade-and-note:** if a `notion-query-data-sources` call in §3 ever
returns an upgrade/plan-gating prompt instead of results, degrade to this
path for that data source only (scoped `notion-search` + `notion-fetch`
covering the same question), and:

- **Note the degradation explicitly** in the final answer — which data
  source fell back and that results may be less precisely filtered/sorted,
  since search+fetch can't apply the same structured filter the structured
  path would have.
- **Surface the decision, don't make it**: tell the user this workspace's
  plan gates structured queries on that data source, and that the choice is
  between upgrading the Notion plan (restores structured filtering) or
  accepting the search+fetch fallback as the permanent path for it — per
  `notion-conventions.md`. Don't silently pick one.

### 5. Answer with citations

Ground every claim in the final answer in content actually returned by a
query or a fetch — a row's real property values, or a page's real body text.
**Never fabricate** a fact, a date, or a status that wasn't actually
observed. Cite each source used, by its page/row title and Notion link,
either inline or as a short "Sources" list at the end of the answer.

If nothing in the wiki, tasks, journal, or inbox actually answers the
question, **say so plainly** — state that nothing was found, rather than
producing a plausible-sounding but ungrounded answer or a hedge that implies
partial knowledge that doesn't exist.

### 6. Optional file-back

Only when the user asks for it (never automatic) — after producing the cited
answer, write it to a destination:

- **Wiki** — `notion-create-pages` parented to the wiki **page** URL
  (`wiki.database_id` from config), never `wiki.data_source_url`, per the
  wiki parent-URL rule in `notion-conventions.md`. Title from the question or
  a user-given title; body = the cited answer, including its source links.
  If instead updating an already-existing wiki page the user points at,
  prefer `insert_content` over a whole-page replacement so concurrent human
  edits aren't clobbered.
- **Journal** — `notion-create-pages` a **distinct new row** in
  `journal.data_source_url`: `Name` a short label for the question/answer,
  `Date` = today (resolved against `preferences.timezone`), body = the cited
  answer. **Leave `Type` unset.** Never upsert into today's `Type = Daily`
  row — that row is owned exclusively by the `today` skill and matched by
  `Date` + `Type = Daily`; a filed answer is a separate dated row with `Type`
  unset, per the Journal invariant in `schema.md`. If the user instead points
  at an existing non-Daily row to append to, prefer `insert_content` there
  rather than creating a duplicate.

### 7. Errors

- **MCP unavailable** (`notion-query-data-sources`, `notion-search`,
  `notion-fetch`, `notion-create-pages`, or `notion-update-page` fails,
  unauthenticated or otherwise): say so plainly, point the user at claude.ai
  → Settings → Connectors → Notion, and **stop** — never fake an answer or a
  citation.
- **Rate limiting (`429`):** honor `Retry-After` and back off before
  retrying, per `notion-conventions.md`.
- **Large/async writes:** if a file-back write returns an async task, poll
  `notion-get-async-task` rather than assuming completion before reporting it
  done.
- **Partial failure** (e.g. one shared space's task query fails but others
  succeed, or one `notion-fetch` fails but other hits succeed): answer with
  whatever grounded content succeeded, and clearly flag which source failed
  and why — don't silently drop it or fail the whole answer for one bad
  source.

## Smoke test

Smoke test (dual-path, both required):
- **Structured path:** query "what tasks are due this week?" → single-source
  queries over each task DB, merged, answer cites the rows.
- **Fallback path:** force/simulate the search+fetch path (e.g. a
  wiki-content question "what did we decide about the venue?") →
  `notion-search` scoped + `notion-fetch` of hits → answer cites the pages.
  If structured querying returns an upgrade prompt, confirm the skill
  degrades to this path and **notes the degradation** in its output.

Assertions: each answer is grounded in fetched Notion content with citations
(page/row titles + links); no fabricated facts; optional file-back writes a
page when asked (a Journal file-back creates a distinct new row with `Type`
unset rather than touching today's `Type = Daily` row; a wiki file-back
parents to the wiki page URL, not its data source).
