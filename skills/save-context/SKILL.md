---
name: save-context
description: Use when the user wants to record durable, workspace-specific context about their Notion second-brain setup — a naming or tagging convention, a custom property worth weighing, a workspace quirk, a partner working preference — so future sessions honor it. Saves a terse bullet to the private second-brain repo's CLAUDE.md (`## Second brain context`) in durable mode, or to the Notion AGENTS page's `## Context` section in ephemeral mode (e.g. Cowork). Use this whenever the user says to "remember", "note", "keep in mind", or "save" something about how their Notion workspace is organized — as opposed to capturing a task/note (that's `capture`) or changing structured config like timezone or task-DB mappings (that's `setup`).
---

# save-context

Record a durable fact about the user's Notion setup so every future session
starts already knowing it. The fact lives as one bullet in a mode-aware home
(see `../shared-references/durability-modes.md`): the **user's private
second-brain repo `CLAUDE.md`** `## Second brain context` section in `durable`
mode, or the AGENTS page's `## Context` section in `ephemeral` mode. Either way
this skill only records saved context; it never touches `config.json`.

Consult `../shared-references/saved-context.md` for the managed section, the
marker pair, what qualifies vs. what to redirect, and the append-only
create-or-update protocol. Consult `../shared-references/schema.md` for the
`config.json` key set — the boundary that tells structured config (which belongs
in `config.json` via `setup`) apart from soft context (which belongs here).

Consult `../shared-references/durability-modes.md` for the mode this skill
branches on — `durable` writes the repo `CLAUDE.md`; `ephemeral` writes the
AGENTS page's `## Context` section.

## Behavior

### 1. Resolve the durability mode and pick the home

Resolve the durability mode (`durability-modes.md`) and pick the home:

- **`durable`** — the launch-folder cwd has a `config.json` (the established-repo
  signal, per `setup` §0.0), or a `CLAUDE.md` already carrying the
  `<!-- ns2b:context:start -->` marker. Write to that repo `CLAUDE.md`'s
  `## Second brain context` section, exactly as before. Never write the section
  into an unrelated `CLAUDE.md`.
- **`ephemeral`** — no local config/repo. Resolve config from the AGENTS page
  (the discovery fallback: `notion-search` the `Second Brain` root →
  `notion-fetch` its `AGENTS` page → read the fenced config block, per
  `schema.md`). The saved-context home is that page's `## Context` section.
- **Neither resolvable** (no repo signal *and* no discoverable AGENTS config):
  **stop** and say so plainly — tell the user to run this from their second-brain
  repo (`cd <repo> && claude`) or run `setup` first. Never guess at IDs.

### 2. Get the fact

Take it from the invocation text if the user gave one (e.g. `save-context "the
Wiki's Evergreen tag means never expire"`). If they invoked the skill with
nothing to save, ask what they'd like to record. Never invent a fact.

### 3. Validate — is this saved context, or something else?

Saved context is a **durable, workspace-specific fact or preference not already
in `config.json`** (per `saved-context.md`): a naming/tagging convention, a
custom property worth weighing, a workspace quirk, a partner working preference.

Redirect anything that isn't:

- **A task, note, idea, or reference** ("call the plumber Friday") → that's
  `capture`, not context. Say so and point the user there — offer to hand it to
  `capture` instead.
- **Structured config** — a timezone, a task-DB role mapping or status value, a
  member ID, a page/DB ID → that lives in `config.json` (and its AGENTS mirror),
  written by `setup`. Say so and point the user at `setup`. Timezone is the
  common trap: it belongs in `preferences.timezone`, never a context bullet.

If it's genuinely ambiguous, ask one clarifying question rather than guessing.

### 4. Write the bullet

Follow the append-only protocol in `saved-context.md`: locate the repo
`CLAUDE.md`; find the region by the marker pair → else by the
`## Second brain context` heading → else create the section (with markers) after
the `## Second brain repo` block; append the new bullet on its own line **just
before** the end marker. Preserve every existing bullet and any hand-edits
between the markers — never rewrite the section wholesale. Dedupe conservatively:
skip only a near-exact restatement; when unsure, append. Do not touch
`config.json`.

**Ephemeral mode:** instead of the repo file, append the bullet to the AGENTS
page's `## Context` section via `insert_content` (create the section if absent),
per `saved-context.md`'s ephemeral home. Same append-only, conservative-dedupe
rules. Do not write any local file.

### 5. Confirm

Echo the exact bullet you saved and where — `CLAUDE.md` → `## Second brain
context` in `durable` mode, or the AGENTS page's `## Context` section in
`ephemeral` mode. In `durable` mode with `git` present, note that the auto-commit
Stop hook will sync it at session end (no manual commit needed); if `git` is
absent (`durability-modes.md`), tell the user to commit the change from their own
machine instead. In `ephemeral` mode the AGENTS write is already durable in
Notion — nothing to commit. If nothing was written because the fact was
redirected in §3, say which skill to use instead.

### 6. Non-interactive / errors

- **No fact and no one to ask** (invoked non-interactively with empty input): do
  not guess — report that there was nothing to save.
- **Not in a second-brain repo** (§1 guard failed): stop and give the `cd`/`setup`
  guidance above; never fabricate the location.

## Smoke test

Smoke test (run inside a live second-brain repo):
- `save-context "the Wiki uses a Status = Evergreen tag for pages that never expire"`
  → a matching bullet is appended inside the `ns2b:context` marker pair in the
    repo's `CLAUDE.md`; `config.json` is untouched; the response echoes the saved
    bullet.
- `save-context "call the plumber about the leak on Friday"`
  → not saved; redirected to `capture` (it's a task, not context).
- `save-context "my timezone is America/Chicago"`
  → not saved; redirected to `setup` (structured config → `preferences.timezone`).
- Hand-add a bullet inside the markers, then `save-context "…"` again
  → the hand-added bullet survives and the new one is appended (no wholesale
    rewrite, no duplicate).
- **Ephemeral (run live in Cowork — not confirmable from the CLI):** with no
  local `config.json`/`CLAUDE.md` and a non-`cli` entrypoint, `save-context
  "…"` resolves config from the AGENTS page and appends the bullet to that
  page's `## Context` section via `insert_content` (no repo write, no error).
- **No-git note:** in a durable repo without `git`, the confirm message tells the
  user to commit from their own machine rather than claiming the Stop hook synced
  it.

Assertions: after a real save, `CLAUDE.md` has exactly one new bullet inside the
markers, existing bullets/hand-edits are intact, and `config.json` is unchanged;
task-like and config-like inputs are redirected rather than written. In
`ephemeral` mode the bullet lands in the AGENTS `## Context` section (not a
repo file); when `git` is absent the confirm message degrades the Stop-hook
claim.
