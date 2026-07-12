# Task DB Mapping Contract

## Purpose

Task databases (personal and shared) are **discovered and mapped, never
mutated**. The plugin no longer forces a canonical schema onto a user's
existing task DB — it records, per DB, which real property holds each
semantic **role** (title, status, due, …) and which real status value plays
each **status role** (open default, done, waiting). Every skill that reads or
writes a task resolves names through this mapping instead of assuming
`Name`/`Status`/`Due` etc. This lets personal and shared task DBs keep
whatever shape they already have — including two DBs with completely
different property names — while every skill uses one code path.

## Role catalog

**Mandatory roles** (setup fails loudly if it can't find *any* candidate for
these; if a candidate exists but is ambiguous, it is recorded under
`unconfirmed_roles` like any other role — see Confirmed vs unconfirmed
mappings below — not treated as a failure):

| Role | Meaning |
|---|---|
| `title` | the title property |
| `status` | the status/select property driving open vs done |

**Optional roles** (omitted from `properties` when absent; skills skip any
filter/write for an unmapped role — never substitute a canonical name):

| Role | Meaning |
|---|---|
| `due` | hard deadline — a date that must not be missed |
| `scheduled` | do/work date — when the task is planned to be worked, not when it's due |
| `assignee` | person responsible (shared DBs) |
| `priority` | priority select/field |
| `project` | project/grouping select or relation |
| `source` | capture provenance (url or rich_text) |

`due` and `scheduled` are independent and both optional — a DB may have
either, both, or neither. Example: the home-renovation DB has both (`due:Deadline`,
`scheduled:Do date`); a personal DB with a single date property maps it to
`due` and omits `scheduled`.

**Optional status-value role:**

| Role | Meaning |
|---|---|
| `waiting` | the status value meaning "blocked, awaiting the other person" |

`waiting` is the status value `two-person-rules.md`'s "blocked on your
partner" behavior depends on. If a DB has no equivalent status value, skills
**skip** the waiting-specific behavior and **note** that the DB has no
waiting status — never invent or add one.

## Skill resolution rule

**Look up role in the active DB's `properties`; use the returned real name;
if absent, skip — never fall back to a canonical name.**

## Status handling

**Write `open_default`; filter `status ∉ done[]`; never hard-code `Done`.**

## Confirmed vs unconfirmed mappings (non-interactive contract)

Each task-DB mapping object carries a confirmation marker: either
`"confirmed": true`, or `"unconfirmed_roles": [...]` listing the roles that
are still ambiguous.

- When `setup` can prompt (interactive session), it disambiguates any
  ambiguous role with the user and records a **confirmed** mapping.
- When `setup` **cannot** prompt (Routine/autonomous run), it runs inference,
  records the mapping with its ambiguous roles listed under
  `unconfirmed_roles`, and reports them as pending picks. **It never guesses
  an ambiguous role.**
- Identity mappings (a freshly-scaffolded personal DB, property names chosen
  by the plugin itself) are `confirmed` by construction — there is nothing to
  disambiguate.
- This applies uniformly to mandatory and optional roles: mandatory-ness only
  changes what happens when *no* candidate exists at all (fail loudly, per
  Role catalog above) — it does not exempt a mandatory role from landing in
  `unconfirmed_roles` when a candidate is found but ambiguous.

**Resolution rule addition:** If the active DB's mapping is unconfirmed for a
role the operation needs, refuse that operation and report that setup must
confirm the mapping — never operate on an unconfirmed role.

## Setup derivation heuristics

Used by `setup` (and referenced, not duplicated, by its SKILL.md) when
discovering/adopting an existing task DB.

**Property → role inference:**

| Signal | Inferred role | Ambiguity handling |
|---|---|---|
| title-typed property | `title` | none — always unambiguous (exactly one per DB) |
| status-typed property | `status` | none |
| select property named like "Status"/"State" (no status-typed property exists) | `status` (candidate) | ask which select is the status driver |
| date property | `due` or `scheduled` | one date prop → `due`; multiple date props → ask which is the hard deadline (`due`) vs. the do/work date (`scheduled`) |
| person property | `assignee` | ask if multiple person properties exist |
| property named/typed like "Priority" | `priority` | name+type match; omit if absent |
| property named/typed like "Project" | `project` | name+type match; omit if absent |
| property named/typed like "Source" | `source` | name+type match; omit if absent |

**`status_values` derivation from Status groups:**

- If the status property is Notion's native `status` type, it has three
  built-in groups (To-do / In progress / Complete):
  - every value in the **To-do** group → candidate for `open_default` (ask if
    more than one To-do value exists)
  - every value in the **Complete** group → `done[]`
  - a value in the **In progress** group meaning "blocked/awaiting the other
    person" (e.g. "Awaiting response") → `waiting`, if one exists
- If the status role was filled by a plain **select** (not native `status`
  type), there are no groups — ask the user which values count as "done" and
  which (if any) count as "open default" / "waiting."
- Always echo the derived `status_values` and confirm before writing
  (interactive); in Routine/autonomous mode, auto-infer and flag any
  ambiguous grouping via `unconfirmed_roles`.

## Worked examples (canonical)

Personal task DB, identity-mapped (scaffolded by the plugin, so `confirmed`
by construction):

```json
"tasks_personal": {
  "data_source_url": "collection://…",
  "confirmed": true,
  "properties": { "title":"Name","status":"Status","due":"Due","assignee":"Assignee","priority":"Priority","project":"Project","source":"Source" },
  "status_values": { "open_default":"Not started", "done":["Done","Archived"] }
}
```

Shared task DB, adopted from an existing home-renovation-planning database with its
own names and status set:

```json
"shared_spaces":[{ "name":"…","root":"…","members":[…],
  "tasks":{ "data_source_url":"collection://…",
    "confirmed": true,
    "properties":{ "title":"Task","status":"Status","due":"Deadline","scheduled":"Do date","assignee":"Assignee","priority":"Priority" },
    "status_values":{ "open_default":"Not started","done":["Will not do","Done"],"waiting":"Awaiting response" } } }]
```

Note this shared example has no `project`/`source` roles — both are omitted
from `properties` (never written as `null`), and skills skip any
`project`/`source` filter or write against this DB.

## Timezone

Resolve all dates against `preferences.timezone`; never trust the running
session's clock — it is UTC in cloud/Routine/Cowork sessions.
