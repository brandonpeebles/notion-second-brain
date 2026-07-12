# Notion Query Plan-Gating

The hosted Notion MCP (`mcp.notion.com` / the claude.ai Notion connector)
plan-gates its query tools. This is an expected, documented boundary — **not a
bug and not a crash.** Any query-dependent skill (`query`, `today`, `triage`,
`setup`, and any future cross-DB skill) imports this file so the gate is
detected and handled the same way everywhere.

Verified via live tests 2026-07-12; encode here so it's never re-derived.

## The gate in one line

`notion-query-data-sources` and `notion-query-database-view` require **Business
+ Notion AI** for a *single* data source, and **Enterprise + Notion AI** to
query *2+ data sources in one call*. On lower tiers the tool **throws** — it
does not return a soft "upgrade" payload you can inspect in the results.

## Tier map (verified)

| Query shape | Plus | Business + Notion AI | Enterprise + Notion AI |
|---|---|---|---|
| Single-source filtered/sorted query (1 data source) | ❌ gated | ✅ works | ✅ works |
| View-mode query (`notion-query-database-view`) | ❌ gated | ✅ works | ✅ works |
| Cross-source query (2+ data sources, one call) | ❌ gated | ❌ **gated** | ✅ works |

Ungated on **every** tier (incl. Plus/Free), so they are the fallback surface:
`notion-fetch`, `notion-search`, all create/update tools, and the raw Public
API (see "Escape hatch" below).

Business + Notion AI is the reference tier these skills target: **single-source
queries work, cross-source does not.** That's why the standing rule is one
single-source query per data source, merged in the skill layer — never a
cross-source call. Decomposition keeps you inside the gate by construction, so
on this tier the error below should essentially never fire from the skills' own
calls; it's the safety net for an unexpected gate (e.g. a downgraded workspace,
or a Plus-tier user running the skill).

## The exact error signature (what to detect)

The gate surfaces as a **thrown `APIResponseError`**, not a result payload —
code that only inspects the returned rows for an "upgrade prompt" will miss it.
The cross-source variant:

```
status: 400
code: "validation_error"
message: "Querying across multiple data sources requires an Enterprise plan with
          Notion AI. Query a single data source, or upgrade your plan to query
          multiple at once:
          https://app.notion.com/checkout?source=mcp_tool_upsell&tool=query_data_sources&product=enterprise"
```

Treat a query error as this class of event — the **plan gate** — when any of:

- `code === "validation_error"` **and** the message mentions a plan
  (`"requires an Enterprise plan"`, `"requires a Business"`, or similar
  plan-gate language), **or**
- the message/URL contains `source=mcp_tool_upsell`.

The Plus-tier variant (single-source blocked) surfaces a sibling plan-gate
message; the same heuristic catches it. Distinguish this from a **generic 400**
(a malformed filter, an unknown property) — those are real bugs to fix, not the
plan gate, and must not be swallowed as "just the paywall."

## Decision tree

1. **Prefer decomposition — stay ungated by construction.** A task spanning 2+
   data sources runs **one single-source query per source**, merged/joined in
   the skill's own context. This is the standing rule for these skills; only a
   genuine DB-side cross-source aggregate/anti-join over large data (see below)
   would ever need a single multi-source call, and that path is deferred.
2. **On a plan-gate error — do not retry the same call.** Recognize the
   signature above, then, for the gated data source only:
   - Fall back to the ungated path — scoped `notion-search` + `notion-fetch`
     covering the same question — if the intent allows it.
   - **Note the degradation** in the output: which data source fell back, and
     that search+fetch can't apply the same structured filter/sort, so results
     are less precisely filtered.
3. **Surface the decision, don't make it.** Tell the user this workspace's plan
   gates structured queries on that data source, and that the choice is between
   upgrading the Notion plan (restores structured filtering) and accepting the
   search+fetch fallback as the permanent path. Don't silently pick one.
4. **Never present the gate as a failure/crash.** It's a plan boundary — name it
   as one.

## When a cross-source query is genuinely needed (background)

Merge-in-context covers essentially everything at personal/household scale. A
true DB-side cross-source query is only needed when merging breaks: both sides
large, a DB-side aggregate/GROUP BY across sources, or an anti-join ("rows in A
with no link to B" — the "gap audit" pattern). The real trigger is an
**automated, unattended (Cowork) gap-audit/aggregate over large accumulated
DBs** — likely a year+ out. Until then, decomposition is the answer.

## Escape hatch (raw Public API) — deferred, pointer only

The raw Notion **Public API** `POST /v1/data_sources/{id}/query` is **not**
behind this gate (works on all plans, incl. Plus/Free). But a local script is
unreachable from Claude cloud / Cowork / mobile — to be usable there it must be
exposed as a **remote MCP custom connector**. Build this only if a real
automated, unattended cross-DB job appears; until then it's a pointer, not a
path.
