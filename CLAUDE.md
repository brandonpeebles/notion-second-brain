# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **Claude Code plugin** whose entire product is prompt/instruction Markdown — there is no application code, no build step, no test runner, no lint. The plugin runs a "second brain" on Notion (capture, daily brief, inbox triage, cited query) entirely through the built-in **Notion connector** (hosted MCP, `notion-*` tools). No MCP config files or hooks are wired up by the plugin.

Tested surface is the **Claude Code CLI with the claude.ai Notion connector**; other surfaces (Cowork, claude.ai chat/mobile, cloud, Routines) are supported in principle but unverified (see README "Install").

## Layout

- `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` — plugin + single-plugin marketplace manifests.
- `skills/<name>/SKILL.md` — the seven skills: `setup`, `capture`, `today`, `triage`, `query`, `save-context`, `cowork-context`. Each has YAML frontmatter (`name`, `description`) and a `## Behavior` section of numbered steps, plus (except triage/query where inline) a `## Smoke test` block. `save-context` maintains a `## Second brain context` section in the *user-repo* `CLAUDE.md`; `cowork-context` compiles config + that context into a pasteable Cowork block.
- `skills/shared-references/*.md` — the canonical contracts every skill imports by relative path. **These are the single source of truth; skills reference them and must not duplicate their content.**
- `config.json` at repo root is a **local sample only** — the real one is written per-user into the launch-folder cwd and is gitignored (`config.json`, `**/config.json`). It holds workspace/user Notion IDs; never commit a real one and never echo those IDs into any tracked file.
- `.superpowers/` and `docs/` hold SDD scratch/specs (gitignored / handoff docs), not shipped plugin content.

## Shared references (read these before touching any skill)

- `schema.md` — canonical DB schemas (Inbox/Journal/Archive), the `config.json` key set, the workspace structure (root → Inbox/Tasks/Wiki/Journal/Archive/Home/AGENTS), and adopt/patch rules.
- `task-db-mapping.md` — the role-mapping contract for task DBs (see invariants below).
- `notion-conventions.md` — live-learned Notion MCP quirks (icons, wiki creation, status defaults, no-trash, async writes, rate limits, connector identity).
- `query-plan-gating.md` — the plan-gate error signature, per-tier map, and dual-path fallback decision tree.
- `two-person-rules.md` — shared-space (partner) routing, assignee, and Waiting semantics.
- `saved-context.md` — the managed `## Second brain context` section in the *user-repo* `CLAUDE.md`: what qualifies as saved context vs. config, the marker pair, and the append-only create/update/repair protocol (used by `save-context`, `setup`, `cowork-context`).

## Non-obvious invariants (violating these is a bug)

- **Two config sources, kept in sync.** State lives in the launch-folder `config.json` *and* is mirrored into the Notion `AGENTS` page's fenced ```json block. On surfaces with no durable filesystem (Cowork), the AGENTS block is the only surviving copy — that's why `setup` reads the mapping from it before re-discovering. Any config change must update both.
- **Skills address databases by `data_source_url` (`collection://…`) from config, never by display name** (except during `setup` discovery).
- **Internal DBs vs. task DBs are handled oppositely.** Inbox/Journal/Archive/Wiki are plugin-owned: their property names are **canonical and fixed** (use exactly as `schema.md` spells them) and `setup` may additively patch them. Task DBs (`tasks_personal`, every `shared_spaces[].tasks`) are **discovered and mapped, never mutated** — no DDL ever. Resolve each role (`title`/`status`/`due`/`scheduled`/`assignee`/`priority`/`project`/`source`) and each status value (`open_default`/`done[]`/`waiting`) through that DB's `properties`/`status_values` mapping; **skip an unmapped role — never fall back to a canonical name like `Name`/`Status`/`Due`.**
- **Single-source queries only.** `notion-query-data-sources` takes one data source; to span personal + shared DBs, query each and merge in the skill. Never a cross-data-source call.
- **Dual-path querying.** Primary: structured `notion-query-data-sources`. The plan gate surfaces as a **thrown `400` `APIResponseError`** (not an inspectable payload) — catch it, detect via `code === "validation_error"` + plan-gate language / `source=mcp_tool_upsell`, fall back to scoped `notion-search` + `notion-fetch`, and surface the upgrade-vs-fallback decision rather than silently picking. Distinguish from a generic 400 (a real bug — don't swallow it as "the paywall").
- **Timezone comes from `preferences.timezone`, never the session clock** (which is UTC in cloud/Routine/Cowork).
- **Native Notion icons, plain titles.** Never embed an emoji in a page/DB title — an emoji in the title *and* the icon slot renders two icons. On adopt, repair: set native icon, strip leading emoji.
- **No page-trash tool.** Discard by moving rows into the `Archive` DB with `notion-move-pages`.
- **Wikis, the Waiting status option, and the Notion AI instructions page are UI-only** — cannot be created/set via MCP. `setup` guides the user and verifies, or reports them as pending in non-interactive/Routine mode.
- **Routine/non-interactive mode never prompts or guesses.** Ambiguous task-DB roles are recorded under `unconfirmed_roles` (and an ambiguous personal-DB pick as `{"pending_selection": true, "candidates": [...]}`); skills refuse to operate on an unconfirmed role.

## Working in this repo

- **Development uses the superpowers workflow** (brainstorming → writing-plans → subagent-driven-development, with code review). Skills are edited as prose; there is nothing to compile.
- **Verification is manual and live**: run the affected skill against a real Notion workspace and check its `## Smoke test` assertions. There are no automated tests.
- When you change a rule, change it in the relevant `shared-references/*.md` file and confirm the skills still *point to* it rather than restating it — divergence between a reference and the skills that import it is the main failure mode here.
- Skills are invoked as `/notion-second-brain:<skill>` (e.g. `/notion-second-brain:setup`).

## Cutting a release

**Pushing to `main` is the actual publish** — `marketplace.json` `source: "./"` makes the repo its own marketplace, and installs fetch the default-branch HEAD. The version bump, tag, and GitHub Release are the **explicit version record**, not the delivery step.

**When.** Cut a release when shipping a user-facing change to the skills or shared references. Bump `version` in `.claude-plugin/plugin.json` by user impact:
- **MAJOR** — breaks a contract users depend on: `config.json` shape, `schema.md` DB schemas, or the task-DB role-mapping contract; removing/renaming a skill; anything that breaks an adopted workspace or forces a re-`setup`.
- **MINOR** — additive capability: a new skill, a new behavior, an additive schema patch.
- **PATCH** — prose/behavior fixes and doc corrections with no contract change.

Before cutting, it's recommended (not required — there is no CI) to run the changed skills' `## Smoke test` assertions live against a real Notion workspace.

**How** (worked `vX.Y.Z` example):
1. (Recommended) Run the changed skills' `## Smoke test` and confirm the assertions.
2. Bump `version` in `.claude-plugin/plugin.json` — **the only place the version lives** (`marketplace.json` and SKILL frontmatter carry none, so nothing else to sync).
3. Commit (Conventional Commits; trailers auto-append), e.g. `chore: release vX.Y.Z`.
4. Push `main`: `git push` — this is the actual publish to users.
5. Tag and push it; the tag **must** match `plugin.json` (`v` + version):
   ```bash
   git tag vX.Y.Z
   git push origin vX.Y.Z
   ```
6. Create the GitHub Release from the tag (no CHANGELOG — the Release body is the record):
   ```bash
   gh release create vX.Y.Z --title vX.Y.Z --notes "…summary of user-facing changes…"
   ```

No tags exist yet — `v0.1.0` would be the first.
