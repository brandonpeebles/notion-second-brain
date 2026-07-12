---
name: capture
description: Use when the user wants to quickly capture a task, note, idea, or reference into the second brain with zero decisions — routes to the Inbox, or straight to Tasks when unambiguous. Phone-first; echoes parsed natural-language dates back for confirmation.
---

# capture

Zero-decision capture of a task, note, idea, or reference into the second
brain. Phone-first: minimize prompts, default to the Inbox, and only ask a
question when a date is genuinely ambiguous or a shared space is implied but
unresolved.

Consult `../shared-references/schema.md` for the Inbox and Task schemas
(property names/types) and the `config.json` key set. Consult
`../shared-references/two-person-rules.md` for shared-vs-personal routing and
partner resolution. Consult `../shared-references/notion-conventions.md` for
MCP quirks (rate limits, connector identity). Use the config keys and
property names exactly as `schema.md` defines them — do not paraphrase or
rename.

## Behavior

### 1. Load config or discover

Look for `config.json` in the launch folder. If present and valid, read
`inbox.data_source_url`, `tasks_personal.data_source_url`,
`shared_spaces[].tasks` (and `shared_spaces[].members`), and
`preferences.timezone` from it.

If no `config.json` is found, fall back to discovery: `notion-search` for the
root page named exactly `Second Brain` (emoji prefix allowed), then
`notion-fetch` its `Home` child page and read the fenced ```json config
block for the same keys, per the discovery convention in `schema.md`.

If neither resolves (no config file and discovery finds no root/Home config
block, or finds multiple ambiguous candidates): **fail loudly**. State
plainly that the second brain isn't set up yet, point the user at the
`setup` skill, and **stop**. Never guess at IDs or invent a target.

### 2. Choose the target: Inbox (default) or Tasks (shortcut)

**Default target is the Inbox.** Unless the straight-to-Tasks shortcut below
applies, create a row in `inbox.data_source_url` with `notion-create-pages`:

- `Name` — the captured text, lightly cleaned (trim filler like "capture" or
  "remind me to").
- `Type` — best guess among `Task` / `Note` / `Idea` / `Reference` (per
  `schema.md`).
- `Source` — a URL or short origin text if the input includes one; otherwise
  leave blank.
- `Triage` — set explicitly to `New`.
- `Captured` — automatic (`created_time`); never set it manually.

**Straight-to-Tasks shortcut:** when the input is unambiguously a task —
imperative phrasing plus a due date, or the user explicitly says "task" —
create the row directly in the Tasks data source (personal or shared, per
routing below) instead of the Inbox, with `notion-create-pages`:

- `Name` — the captured text, cleaned.
- `Status` — `Not started`.
- `Due` — the parsed date (see NL dates below).
- `Source` — a URL or short origin text if present; otherwise leave blank.
- `Assignee` — required whenever the row is routed to a shared space; set it
  from the named partner, or ask/flag when none is named (see routing below).
  Never routed to a shared space → leave unset.

If the input doesn't clearly meet the shortcut bar, default to the Inbox —
never guess a Task classification to avoid a follow-up question.

### 3. Natural-language dates

Parse any relative date ("Friday", "next Tuesday", "in two weeks") against
`preferences.timezone` into an absolute date. Always **echo the resolved
date back** in the response, e.g. `"Friday → 2026-07-17"`, so the user can
see what was written.

If a date is genuinely ambiguous (more than one reasonable reading) and the
session is interactive, ask which one before writing. Never silently guess
an ambiguous date. In Routine mode, see §5 — there is no one to ask.

### 4. Shared routing

Per `two-person-rules.md`: route to a shared space's Tasks database
(`shared_spaces[].tasks.data_source_url`) only when that space is named or
clearly implied by the input. Otherwise capture is personal — use
`tasks_personal.data_source_url` (for the Tasks shortcut) or the personal
`inbox.data_source_url` (default path). When ambiguous, default personal.

When the input says "assign to <partner>" (or similar) and the target is a
shared space: resolve `<partner>` against that space's `members` list from
config, then confirm/resolve the Notion user via `notion-get-users`, and set
`Assignee` to that person. If `<partner>` doesn't match a member of that
space, say so plainly rather than guessing an ID, and leave `Assignee`
unset.

**Shared tasks always get an Assignee** (`two-person-rules.md`: a shared-space
task with no assignee is incomplete — ask or infer, don't leave it blank). So
when the row is routed to a shared space's Tasks DB but no partner is named:
in interactive mode, ask who it's for before writing; in Routine mode (§5),
mirror the ambiguous-date pattern — leave `Assignee` unset, still create the
row, and flag the missing assignee in the report rather than guessing.

Never write into a partner's private area — shared writes only ever target
the named shared space's own Tasks database.

### 5. Routine mode (non-interactive)

If invoked non-interactively (no way to prompt for confirmation): create the
Inbox (or Tasks) row without prompting, using the best resolution available.
Report the resolved date and any routing decisions in the response instead
of asking to confirm. If a date is genuinely ambiguous and there's no one to
ask, leave `Due` unset, still create the row, and flag the ambiguity in the
report rather than guessing. Apply the same pattern to a missing shared-space
`Assignee` (§4): leave it unset, still create the row, and flag it.

### 6. Errors

- **MCP unavailable** (`notion-create-pages`, `notion-get-users`,
  `notion-search`, or `notion-fetch` fails, unauthenticated or otherwise):
  say so plainly, point the user at claude.ai → Settings → Connectors →
  Notion, and **stop**. Never fake success.
- **Rate limiting (`429`):** honor `Retry-After` and back off before
  retrying, per `notion-conventions.md`.

## Smoke test

Smoke test (run against a live Notion workspace):
- `capture "call the plumber about the leak, due Friday"`
  → creates an Inbox row: `Name` set, `Type` = `Task` (guess), `Source`
    blank/text, `Triage` = `New`, `Captured` auto; echoes the resolved date
    ("Friday → 2026-07-17") back for confirmation before/at write.
- `capture "add 'buy anniversary gift' to the tasks for <a named shared
  space>, assign my partner"`
  → routes to that shared space's Tasks DB (since it's named), `Assignee` =
    partner resolved from that space's members; `Status` defaults to
    `Not started`.

Assertions: rows appear in Notion with the stated properties; the
natural-language date is echoed and correct in the configured timezone.
