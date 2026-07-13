---
name: cowork-context
description: Use when the user wants to run their Notion second brain in Cowork (or another surface with no durable filesystem) and needs the context to paste into Cowork's project-instructions field. Emits a thin pointer to the workspace's AGENTS page (its URL/ID plus a one-line instruction to read config and saved context there) — not a full config dump — prints it, and copies it to the clipboard for manual paste. Use whenever the user mentions Cowork, "project instructions", pasting their second-brain setup into another Claude surface, or asks how to make their second brain work outside the local CLI.
---

# cowork-context

Cowork projects have a project-instructions field, but a Cowork session has no
durable filesystem — it can't read the local `config.json` or `CLAUDE.md`.
Because the workspace config (and, in ephemeral mode, saved context) already
lives on the Notion **AGENTS page**, this skill doesn't re-dump it. It emits a
thin **pointer** — the AGENTS page URL/ID plus a one-line instruction to read
config and the `## Context` section there — prints it, and copies it to the
clipboard so the user can paste it into their Cowork project's instructions
field. Pasting is a manual step; it can't be automated.

The block is a **pointer, not a payload** — it assumes the `notion-second-brain`
skills are installed in Cowork and own all behavior; it just steers the session
to the one Notion page that holds the durable state. Deliberately, it does **not**
inline `config.json`, restate the skills' steps, the `schema.md` tables, or the
icon/no-trash/async conventions. Replacing the per-session root name-search with a
single deterministic `notion-fetch` of the AGENTS page is the whole point.

Consult `../shared-references/schema.md` for the `config.json` block shape and the
discovery fallback (reading config from the AGENTS page when there's no local
file). Consult `../shared-references/saved-context.md` for the saved-context homes the
pointer directs the session to read — `## Second brain context` in the repo
`CLAUDE.md` when `durable`, the AGENTS page's `## Context` section when
`ephemeral`.

## Behavior

### 1. Resolve config

Same pattern as the other skills: read `config.json` from the launch folder if
present and valid. If there's no local file, fall back to discovery —
`notion-search` for the `Second Brain` root, `notion-fetch` its `AGENTS` page,
and read the fenced ```json config block (per `schema.md`). If neither resolves,
**fail loudly**: say the second brain isn't set up yet, point the user at
`setup`, and **stop**. Never guess at IDs.

### 2. Saved context (no longer inlined)

No longer inlined. Saved context lives on the AGENTS page: in `ephemeral` mode in
that page's `## Context` section, in `durable` mode in the repo `CLAUDE.md` the
user still has locally. The pointer block (§3) tells the session to read
`## Context` from the AGENTS page, so this skill does not copy those bullets into
the block.

### 3. Compile the pointer block

Assemble exactly this structure. Fill in the AGENTS page URL and ID from the
resolved config (`agents_page`). Keep it thin — a pointer, no config contents,
no schema tables:

```markdown
# Notion Second Brain — Cowork pointer

Context for the installed notion-second-brain skills — do not re-derive their
behavior. This session can't read local files; the durable config and saved
context live on the workspace's AGENTS page. Read them from there.

## Where the state lives
- AGENTS page: <agents_page URL> (id: <agents_page id>)
- On first use, `notion-fetch` that page: read the fenced ```json config block
  for all database/page IDs and the task-DB mappings, and the `## Context`
  section for saved workspace conventions.

## Reminders (Cowork gets these wrong)
- Timezone comes from `preferences.timezone` in the AGENTS config block — never
  the session clock, which is UTC in Cowork.
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
as a committed file — it's a pointer to live workspace state on the AGENTS
page, and a committed copy would silently go stale if those IDs change.
Regenerate on demand
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
  → prints one fenced Markdown block containing the **AGENTS page URL/ID** and a
    one-line instruction to read config and `## Context` from that page, plus the
    three Cowork reminders. It does **not** inline the `config.json` contents or
    any `schema.md` tables. The same text is on the clipboard (verify with
    `pbpaste`). No file is written into the repo.
- Temporarily move/rename `config.json`, then `cowork-context`
  → it resolves config from the AGENTS page's config block instead and still
    produces the block.
- With neither a local `config.json` nor a discoverable AGENTS config block
  → it fails loudly and points at `setup` rather than emitting an empty block.

Assertions: the printed block and the clipboard contents match; the block carries
the resolved AGENTS page URL/ID (not a placeholder, not the full config dump);
the three Cowork reminders appear; nothing is committed to the repo.
