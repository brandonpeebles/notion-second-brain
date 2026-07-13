# Saved Context

Durable, workspace-specific context about the user's Notion setup — the facts a
skill would otherwise have to relearn every session — has a **mode-aware home**
(see `durability-modes.md`):

- **`durable` mode** → a managed section of the user's private second-brain repo
  `CLAUDE.md`. Exactly as before: `save-context` writes it, `setup` seeds and
  repairs it, `cowork-context` reads it.
- **`ephemeral` mode** (Cowork with no attached repo, cloud/Routine) → there is
  no repo `CLAUDE.md`, so context lives in an append-only `## Context` section on
  the Notion **AGENTS page**, written via `insert_content`.

One home per mode — **not** a live mirror. The durable and ephemeral homes are
reconciled once, at durable adopt (below), rather than kept in continuous sync;
a continuous Notion↔repo mirror was rejected because it adds a sync obligation
and drift risk with no runtime consumer.

## The managed section

In the user-repo `CLAUDE.md`, a section headed exactly `## Second brain context`,
its body delimited by a start/end marker pair:

```markdown
## Second brain context
<!-- ns2b:context:start -->
<!-- Durable, workspace-specific context the plugin has learned. Managed by
     /notion-second-brain:save-context; hand-edits here are preserved. -->
- Wiki uses a `Status = Evergreen` tag for pages that never expire.
- Inbox has a custom `Energy` select worth weighing during triage.
<!-- ns2b:context:end -->
```

Everything between `<!-- ns2b:context:start -->` and `<!-- ns2b:context:end -->`
is user-owned. Bullets are terse — one fact each.

## The ephemeral home (AGENTS `## Context`)

In `ephemeral` mode there is no repo `CLAUDE.md`. Saved context lives in a
`## Context` section on the **AGENTS page** (the same page that carries the
fenced config block), appended via `insert_content` — the concurrency-safe write
`setup` §6 already mandates for the config block:

```markdown
## Context
- Wiki uses a `Status = Evergreen` tag for pages that never expire.
- Inbox has a custom `Energy` select worth weighing during triage.
```

Same rules as the durable home: terse bullets, one fact each; append-only;
conservative dedupe. Read back by fetching the AGENTS page. No marker pair is
needed — the `## Context` heading delimits it, and the whole page is
plugin-managed.

## Reconciliation at durable adopt

When `setup` first adopts into a `durable` repo and an AGENTS `## Context`
section exists (the user had been running ephemeral, then attached a repo),
append those bullets into `CLAUDE.md`'s `## Second brain context` section
(conservative dedupe, preserve hand-edits) so the durable home is complete. This
is a one-time reconcile, not an ongoing mirror. **Known minor edge (accepted for
v1):** a user who bounces durable→ephemeral→durable may add a bullet in one home
that the other mode doesn't see until the next adopt-reconcile.

## What qualifies (and what does not)

**Save** durable, workspace-specific facts and preferences that are **not** already
captured by `config.json`:

- Custom properties worth weighing (e.g. an `Energy` or `Context` select on a DB).
- Naming or tagging conventions the user follows (e.g. an `Evergreen` wiki tag).
- Workspace quirks the plugin should respect.
- Partner working preferences (e.g. "your partner prefers tasks assigned by default").

**Do not save here — redirect instead:**

- **Transient content** — an actual task, note, idea, or reference → that's
  `capture`, not saved context.
- **Structured config** — timezone, DB role mappings, status values, member IDs,
  page/DB IDs → that lives in `config.json` (and its AGENTS mirror), written by
  `setup`. Timezone is the common trap: it belongs in `preferences.timezone`,
  never a context bullet.

If an input is really one of those, say so plainly and point the user at the right
skill rather than writing a bullet.

## Create-or-update protocol (append-only)

1. **Locate `CLAUDE.md`** in the second-brain repo — the launch-folder cwd, else
   the repo root.
2. **Find the region:** by the marker pair → else by the `## Second brain context`
   heading → else create the section (with markers) immediately after the fixed
   `## Second brain repo` block, or at end of file if that block is absent.
3. **Append** the new bullet on its own line **just before the end marker**. Never
   rewrite the whole section — hand-edits between the markers must survive
   untouched.
4. **Dedupe conservatively:** skip only a near-exact restatement of an existing
   bullet. When unsure whether two facts are the same, append — a redundant bullet
   is cheaper than silently losing a distinct one. You may lightly refine a bullet's
   wording when updating it, but do not merge two distinct facts into one.

Append-only keeps cross-machine git changes small (the repo is cloned and
auto-committed on several machines), so concurrent sessions rarely conflict.

## Boundaries, privacy, and commits

- **Never touches `config.json`.** Touching config could desync the
  `config.json` ↔ AGENTS-block pair; saved context is a separate channel (its
  home is the repo `CLAUDE.md` in `durable` mode, the AGENTS `## Context` section
  in `ephemeral` mode).
- In `durable` mode the user's **private** second-brain repo is the home; in
  `ephemeral` mode the AGENTS page's `## Context` section is. **Never** write
  saved context into the public plugin repo or any tracked plugin file.
- **No new commit logic.** In `durable` mode with `git`, the auto-commit Stop
  hook plus the repo's commit-frequently policy pick up the `CLAUDE.md` change at
  session end; if `git` is absent the user commits from their own machine (see
  `durability-modes.md`). In `ephemeral` mode the AGENTS write is already durable
  in Notion — nothing to commit.

## Repair (owned by `setup`)

A section can drift — its markers or heading deleted, or the section duplicated by a
merge. On re-run, `setup` ensures exactly one `## Second brain context` section with
an intact marker pair exists, creating it if missing and **collapsing duplicates into
one** (preserving every bullet). This mirrors the adopt/icon-repair idempotency
pattern — nothing already present and correct is disturbed.
