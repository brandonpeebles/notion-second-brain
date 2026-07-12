---
name: cowork-context
description: Use when the user wants to run their Notion second brain in Cowork (or another surface with no durable filesystem) and needs the context to paste into Cowork's project-instructions field. Compiles the workspace config plus saved context plus a few Cowork-specific reminders into one Markdown block, prints it, and copies it to the clipboard for manual paste. Use whenever the user mentions Cowork, "project instructions", pasting their second-brain setup into another Claude surface, or asks how to make their second brain work outside the local CLI.
---

# cowork-context

Cowork projects have a project-instructions field, but a Cowork session has no
durable filesystem — it can't read the local `config.json` or `CLAUDE.md`. This
skill compiles what a Cowork session needs — the workspace config, the saved
context, and a few reminders Cowork specifically gets wrong — into one Markdown
block, prints it, and copies it to the clipboard so the user can paste it into
their Cowork project's instructions field. Pasting is a manual step; it can't be
automated.

The block is **config + context only** — not behavior. It assumes the
`notion-second-brain` skills are installed in Cowork and own all behavior; the
block just hands them the durable state and context the filesystem can't carry.
Deliberately, it does **not** restate the skills' steps, the `schema.md` tables,
or the icon/no-trash/async conventions.

Consult `../shared-references/schema.md` for the `config.json` block shape and the
discovery fallback (reading config from the AGENTS page when there's no local
file). Consult `../shared-references/saved-context.md` for the
`## Second brain context` section this reads.

## Behavior

### 1. Resolve config

Same pattern as the other skills: read `config.json` from the launch folder if
present and valid. If there's no local file, fall back to discovery —
`notion-search` for the `Second Brain` root, `notion-fetch` its `AGENTS` page,
and read the fenced ```json config block (per `schema.md`). If neither resolves,
**fail loudly**: say the second brain isn't set up yet, point the user at
`setup`, and **stop**. Never guess at IDs.

### 2. Read saved context

Read the `## Second brain context` section (between the `ns2b:context` markers)
from the second-brain repo's `CLAUDE.md`, per `saved-context.md`. If the file or
section is absent, just omit that part of the block — it's optional.

### 3. Compile the block

Assemble exactly this structure, filling in the resolved config and saved
context. Keep it thin — no behavior, no schema tables:

```markdown
# Notion Second Brain — Cowork context

Context for the installed notion-second-brain skills — do not re-derive their
behavior. A Cowork session can't read the local files; this is the config and
context it needs.

## Config
​```json
{ …the resolved config.json, verbatim… }
​```

## Second brain context
- …saved-context bullets, if any (omit this heading if none)…

## Reminders (Cowork gets these wrong)
- Timezone comes from `preferences.timezone` in the config above — never the
  session clock, which is UTC in Cowork.
- Query one data source at a time. If a structured query throws the plan gate
  (a 400 `validation_error`), fall back to scoped `notion-search` + `notion-fetch`.
- If config or an ID is missing, fail loudly and ask — never guess IDs.
```

### 4. Deliver it

Print the compiled block fenced so it's easy to copy, **and** copy it to the
clipboard with `pbcopy` (macOS): write the block to a temp file **outside the
repo** (e.g. under the system temp dir) and `pbcopy < that_file`, or pipe the
block straight into `pbcopy`. If `pbcopy` isn't available (non-macOS, or a
headless/Cowork session), skip the copy and just print the block, noting the
clipboard step was skipped.

Then tell the user to paste it into their Cowork project's **instructions**
field, and that this paste is manual. **Do not** write the block into the repo
as a committed file — it's a point-in-time compile of `config.json` +
`CLAUDE.md`, and a committed copy would silently go stale. Regenerate on demand
instead. (The block carries workspace IDs — same private posture as
`config.json`; it's for the user's own Cowork project.)

### 5. Errors

- **MCP unavailable during discovery** (`notion-search`/`notion-fetch` fails while
  falling back to the AGENTS block): say so plainly, check `/mcp` and the
  claude.ai login, and **stop** — don't emit a half-empty block.
- **No config anywhere** (§1): stop and point at `setup`; never fabricate IDs.

## Smoke test

Smoke test (run inside a live second-brain repo):
- `cowork-context`
  → prints one fenced Markdown block containing a `## Config` `json` block (the
    resolved config, with DB role mappings, timezone, and any shared
    spaces/members), the `## Second brain context` bullets (if the section has
    any), and the three Cowork reminders; the same text is on the clipboard
    (verify with `pbpaste`). No `schema.md` tables or skill behavior are
    duplicated, and no file is written into the repo.
- Temporarily move/rename `config.json`, then `cowork-context`
  → it resolves config from the AGENTS page's config block instead and still
    produces the block.
- With neither a local `config.json` nor a discoverable AGENTS config block
  → it fails loudly and points at `setup` rather than emitting an empty block.

Assertions: the printed block and the clipboard contents match; the config is
the live resolved config (not a placeholder); saved-context bullets appear when
present and are omitted cleanly when absent; nothing is committed to the repo.
