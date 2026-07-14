# Durability modes

Where `setup` writes `config.json`, and where `save-context` writes saved
context, depends on whether the current session has a **durable, writable home**
— a real filesystem that survives into the next session. This file is the single
source of truth for that decision. `setup`, `save-context`, and `cowork-context`
import it and must **not** restate the ladder.

## The two modes

- **`durable`** — a persistent, writable home exists: the local CLI, or Cowork
  with the user's second-brain repo folder attached. `config.json` lives on disk
  and is the authoritative config store; the AGENTS page's fenced config block
  mirrors it. This is today's behavior.
- **`ephemeral`** — no durable home: Cowork with no attached repo, or a
  cloud/Routine container whose filesystem does not survive. **No local
  `config.json` is written** — a throwaway file could shadow a fresher AGENTS
  block. The AGENTS page's fenced config block is the *sole* durable config
  store, read back via the discovery fallback every skill already implements.

## Detection ladder (deterministic, first match wins)

Runs at the top of `setup`, before any Step 0 bootstrap. Answers one question:
*"do I have a durable, writable home for `config.json` (+ git)?"* Durability
cannot be **proven** from inside a session, so the ladder resolves to `durable`
only on a positive, trusted signal; every other path is `ephemeral`.

1. **Valid `config.json` in the launch-folder cwd** → **`durable`** (adopt).
   Covers the local CLI re-run and Cowork-with-attached-repo. Unchanged path.
2. **`mnt/.local-plugins` or `mnt/.plugins` present** (cwd-relative) →
   **`ephemeral`**. Desktop Cowork's platform convention — the same signal
   Anthropic's `cowork-plugin-customizer` uses to detect Cowork. Absence does
   **not** prove "not Cowork" (a remote container may lack it).
3. **`CLAUDE_CODE_ENTRYPOINT == "cli"`** → **`durable`**. The only environment
   where a fresh git/GitHub bootstrap (setup Step 0) is known-safe.
4. **Everything else** → **`ephemeral`**. Bias every ambiguity here.

**Why bias to `ephemeral`:** a false `ephemeral` is cheap and self-healing —
config lands on the AGENTS page, readable everywhere, and a later CLI run adopts
it. A false `durable` is the worst case — a `git init` + `gh auth` ceremony
inside an evaporating VM, silently lost.

## Demotion overlay (may only demote, never promote)

Before any Step 0 git/GitHub bootstrap in a `durable` result, verify the cwd is
writable **and** `git` is available. If either fails, drop to `ephemeral` for
that run. This probe may only *demote*: writable + git present does **not** prove
durability (a Cowork VM is both, yet ephemeral), so it never promotes
`ephemeral` → `durable`.

**Interactive upgrade (optional, at most once):** in an ambiguous `ephemeral`
case an interactive `setup` may ask exactly once — *"Is this a folder on your own
machine that persists?"* — before bootstrapping. Non-interactive/Routine mode
never asks and stays `ephemeral`.

## No-git degradation (a `durable` home without `git`)

`durable` does **not** guarantee `git`. Cowork-with-attached-repo is a writable,
persistent folder, but the VM often has no `git`. Any step that commits or relies
on the auto-commit Stop hook must degrade gracefully rather than error: **write
the file, then tell the user "edited in your attached folder — commit from your
own machine later," and continue.** This applies to `setup` §9's commit steps and
`save-context`'s "the Stop hook will sync it" note. (The Step 0 bootstrap itself
is unaffected — the demotion overlay guarantees it runs only when `git` is
present.)

## Config-sync invariant (mode-dependent)

- **`durable`:** a config change must update **both** the local `config.json`
  **and** the AGENTS page's fenced config block — they must never desync.
- **`ephemeral`:** there is no local `config.json`. The AGENTS block is the
  single source and the only thing to update.

**Carve-out — `last_scan_ts` is state, not config.** The email scan window state
(`last_scan_ts`) is **not** subject to this invariant. It lives **only** in the
AGENTS page's separate `## Agent state` fenced block (never in `config.json`,
never in the AGENTS **config** block), in **both** modes. It is runtime state, not
configuration — so a daily scan advancing it never triggers a `config.json`↔config-block
sync and never churns git. See `email.md` for the state contract and `schema.md`
for the block's shape.
