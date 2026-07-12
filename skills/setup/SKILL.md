---
name: setup
description: Use when setting up or repairing the Notion second brain — discovers, scaffolds, and adopts the workspace structure, writes config.json, verifies the Notion connector, runs the query-gating test, and registers shared spaces. Idempotent and additive; safe to re-run to add spaces.
---

# setup

Foundational skill for the Notion second brain. Discovers or scaffolds the
workspace structure, adopts what already exists, writes `config.json`,
verifies the Notion connector, runs the query-gating test, and walks the user
through up to three UI-only manual steps (wiki turn-into-wiki; the Waiting
status option, only when a personal task DB is newly scaffolded; and optionally
setting the AGENTS page as the workspace's Notion AI instructions). Every page
and database it creates gets a plain title plus a native Notion icon — never an
emoji embedded in the title (see Icons in `notion-conventions.md`). Its
successful run produces the live
workspace every other skill (`capture`, `today`, `triage`, `query`) operates
on. Idempotent and additive: re-running never destroys anything, and can be
used to add more shared spaces later.

Consult `../shared-references/schema.md` for the canonical DB schemas,
property names/types, the `config.json` key set, adopt/patch rules, and the
discovery config-block format. Consult `../shared-references/task-db-mapping.md`
for the task-DB mapping contract — role catalog, `status_values`, the
confirmed/unconfirmed non-interactive contract, and the derivation heuristics
this skill executes in §3b/§7. Consult `../shared-references/notion-conventions.md`
for MCP quirks (wiki creation, status defaults, no-trash, single-source
queries, dual-path fallback, rate limits, async writes, connector identity),
and `../shared-references/query-plan-gating.md` for the plan-gate error
signature and per-tier gate map (used by the query-gating test, §8). Consult
`../shared-references/saved-context.md` for the managed `## Second brain context`
section in the user-repo `CLAUDE.md` that this skill seeds (§0.3) and repairs (§2).
`config.json`'s **keys** and the **internal DBs'** (Inbox/Journal/Archive)
property names stay canonical — use them exactly as `schema.md` defines them,
do not paraphrase or rename. **Task-DB property names and status values are
not canonical** — they are discovered per-DB and recorded as a mapping
(`properties`/`status_values`/`confirmed`) per `task-db-mapping.md`; never
force a task DB to match the scaffold shape.

## Behavior

Perform these steps in order. Do not skip ahead if an earlier step fails —
stop and report instead.

### 1. Verify connector

Call `notion-get-users`. On failure (no connector, unauthenticated, or any
MCP error): state plainly that the Notion connector is unavailable or
unauthenticated, tell the user to check `/mcp` and confirm they're logged
into Claude Code with their claude.ai account, and **stop**. Never fake
success or proceed as if the workspace is reachable.

On success, record the current user's id and display name — these become
`notion_user_id` and `user` in `config.json` (§9).

### 0. Establish the local repo (bootstrap)

**Runs after §1** (it needs the Notion display name) and **before §2**. On a
fresh machine, `setup` first gives the workspace a durable home: a dedicated
**private GitHub repo** that becomes the launch folder. `config.json` lives
**committed** inside that repo — safe because the repo is private — so remote
sessions (Cowork, Claude Code on the web) can read the live config from the
clone. A commit-frequently policy plus an auto-commit Stop hook keep the remote
copy fresh.

**0.0 Detect state.** If the current working directory already has a
`config.json`, or a `.claude/settings.json` that enables this plugin, you are
already inside an established second-brain repo → **skip Step 0 entirely** and
continue at §2 (adopt path). Otherwise it is a fresh bootstrap — continue.

**0.1 Choose location + name.**
- **firstname:** first token of the Notion display name recorded in §1 →
  fallback the first token of `git config user.name` → fallback ask the user.
- **location default:** `~/src/<firstname>-notion-second-brain` if `~/src`
  exists, else `~/Documents/<firstname>-notion-second-brain`. The repo **name**
  is the folder's basename (`<firstname>-notion-second-brain`).
- Echo both the folder path and the repo name back, and let the user override
  either. If the target folder already exists and is **non-empty**, do **not**
  clobber it — ask the user for a different path.

**0.2 Create + init.**
```bash
mkdir -p "<location>"
git -C "<location>" init -b main
```

**0.3 Inject the scaffold files.** Write these five files into the new repo with
the Write tool (contents are the templates in "Step 0 templates" below):
`.gitignore`, `.claude/settings.json`, `.claude/hooks/autocommit.sh` (then
`chmod +x "<location>/.claude/hooks/autocommit.sh"`), a personal `README.md`,
and `CLAUDE.md`. For `CLAUDE.md`, produce a short baseline first (a one- or
two-paragraph description of the repo), then append the "CLAUDE.md appended
block" template verbatim.

**0.4 GitHub account + remote.**
- `gh auth status` succeeds → create the private repo and push in one step:
  ```bash
  gh repo create "<name>" --private --source "<location>" --remote origin --push
  ```
- `gh` is installed but not authenticated → walk the user through
  `gh auth login`, then run the `gh repo create …` command above.
- No `gh` / no GitHub account → walk the user through creating a GitHub account
  and a **private** repo named `<name>` on github.com (no README/gitignore
  from GitHub — the repo already has them), then:
  ```bash
  git -C "<location>" remote add origin <the new repo's git URL>
  git -C "<location>" push -u origin main
  ```

**0.5 Install the plugin into the repo.**
```bash
claude plugin install notion-second-brain@notion-second-brain-marketplace --scope project
```
The marketplace is already declared in the committed `.claude/settings.json`
(0.3), so this is non-interactive. **State the honesty caveat to the user:** the
committed plugin config auto-loads in the local CLI and (likely) in Cowork, but
**not** in Claude Code on the web — there they must re-run
`claude plugin install notion-second-brain@notion-second-brain-marketplace` once
per session. Record this in the run report.

**0.6 First commit + push.**
```bash
git -C "<location>" add -A
git -C "<location>" commit -m "chore: bootstrap second brain repo"
git -C "<location>" push -u origin main
```
(If 0.4 already pushed via `gh repo create --push`, this commit just adds any
files written after that push.) The auto-commit Stop hook (0.3) keeps future
sessions synced automatically — but only for sessions started **from
`<location>`**; it cannot fire during *this* run, since the active session's
cwd is wherever the user started `claude`, not `<location>`. That means this
run's own later writes (`config.json` in §9) need an explicit commit — see
§9's closing step.

**0.7 Continue.** Proceed to §2 and run the existing Notion steps as today, with
one difference for this bootstrap run: write `config.json` (§9) into the new
repo **by its absolute path** (`<location>/config.json`), not the process cwd.
When the run finishes, tell the user their second brain now lives in
`<location>`, and future runs are `cd <location> && claude`.

#### Step 0 templates

Reproduce each of these verbatim into the new repo in 0.3.

`.gitignore`:
```
# OS / editor
.DS_Store
*.swp

# Claude Code local-only
.claude/settings.local.json
.superpowers/

# Python
__pycache__/
*.pyc
.venv/
.env

# Node
node_modules/
npm-debug.log*
dist/

# config.json is intentionally committed in this private repo — do not ignore it.
```

`.claude/settings.json`:
```json
{
  "extraKnownMarketplaces": {
    "notion-second-brain-marketplace": {
      "source": { "source": "github", "repo": "brandonpeebles/notion-second-brain" }
    }
  },
  "enabledPlugins": { "notion-second-brain@notion-second-brain-marketplace": true },
  "hooks": {
    "Stop": [{ "hooks": [{ "type": "command",
      "command": "\"$CLAUDE_PROJECT_DIR/.claude/hooks/autocommit.sh\"" }] }]
  }
}
```

`.claude/hooks/autocommit.sh` (make it executable — `chmod +x`):
```bash
#!/usr/bin/env bash
# Auto-commit Stop hook for the personal second-brain repo.
# Idempotent and non-blocking: no-op on a clean tree, commit any changes,
# push only when an origin remote exists, and always exit 0 so it never
# blocks the session. Runs as the Stop hook's own shell (not the Bash tool),
# so it does not collide with any git-guardrails PreToolUse hooks.
set -u

cd "${CLAUDE_PROJECT_DIR:-.}" || exit 0
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || exit 0

if [ -z "$(git status --porcelain)" ]; then
  exit 0
fi

git add -A
git commit -m "chore: auto-sync second brain" >/dev/null 2>&1 || exit 0

if git remote get-url origin >/dev/null 2>&1; then
  git push >/dev/null 2>&1 || true
fi

exit 0
```

`CLAUDE.md` appended block (append after the baseline you generate):
```markdown

## Second brain repo

This is a **personal Notion second-brain** repo — it holds the local state and
`config.json` for the notion-second-brain plugin, and it is **private**.

- **Commit frequently and push after any meaningful change.** An auto-commit
  Stop hook (`.claude/hooks/autocommit.sh`) does this automatically at the end
  of each session; committing as you go is still encouraged.
- **`config.json` is intentionally committed here.** It carries your Notion
  workspace/page IDs. That is safe because this repo is private, and it lets
  remote sessions (Cowork, Claude Code on the web) read the live config from the
  clone. Never copy it into the public plugin repo.

## Second brain context
<!-- ns2b:context:start -->
<!-- Durable, workspace-specific context the plugin has learned. Managed by
     /notion-second-brain:save-context; hand-edits here are preserved. -->
<!-- ns2b:context:end -->
```

Personal `README.md`:
```markdown
# My Notion Second Brain

This is my personal second-brain repo. It holds the local state and
`config.json` for the
[notion-second-brain](https://github.com/brandonpeebles/notion-second-brain)
Claude Code plugin.

Launch it with Claude Code:

    cd <this repo> && claude

The repo is private; `config.json` (my Notion workspace/page IDs) is committed
here on purpose so remote sessions can read it.
```

### 2. Locate or create the root

Look for `config.json` in the launch folder first. If it exists and is
valid, treat it as authoritative and skip straight to verifying its
contents against live Notion in step 3 (adopt path). On a fresh bootstrap,
Step 0 has just created the launch folder and there is no `config.json` yet
(it is written in §9); on an adopt run you are already inside the established
repo, where it exists.

If no `config.json` is found, `notion-search` for a page named exactly
`Second Brain` (a leading emoji in the title is tolerated) that is private to
the current user, per the discovery convention in `schema.md`. If a root is
found this way, also `notion-fetch` its `AGENTS` page for the fenced config
block (§6) — it may be the only surviving mapping record (see below).

- **Exactly one match:** adopt it as the root. Repair its icon/title if needed
  (native icon 🧠, plain title `Second Brain`), per the Icons convention.
- **No match:** offer to create `Second Brain` (`notion-create-pages`, plain
  title + `icon: 🧠`), private to the current user, and use it as the root.
- **Multiple matches:** stop and ask the user which one to use. Never guess.

**Read the stored task-DB mapping on re-run.** Whenever a config source
exists — the launch-folder `config.json`, or (absent that) the AGENTS page's
config block — load each task DB's `properties`/`status_values`/`confirmed`
from it before doing any discovery. A **confirmed** mapping is authoritative:
re-verify the DB still has those properties (a cheap `notion-fetch` schema
check), but **do not re-disambiguate** anything about it. Only run discovery
(§3b for the personal DB, §7 for shared spaces) for a task DB with **no**
stored mapping, or one recorded with `unconfirmed_roles`. This matters most
for cross-session Cowork use, where there's no durable `config.json` — the
AGENTS block is the only place the mapping survives between sessions, and
reading it there is what prevents re-prompting the full disambiguation flow
on every run.

**Ensure the `CLAUDE.md` context section.** In the second-brain repo's
`CLAUDE.md`, ensure exactly one `## Second brain context` section with an intact
`<!-- ns2b:context:start -->`/`<!-- ns2b:context:end -->` marker pair exists —
create it (after the `## Second brain repo` block) if missing, and collapse any
duplicates into one, preserving every bullet, per `saved-context.md`. Step 0.3
seeds it on a fresh bootstrap; this runs on every adopt/re-run so an older repo
(cloned before this section existed, or drifted by a merge) gets it back. It
edits that repo file only — never `config.json` or Notion.

**Offer to save context you learn.** When the user reveals a durable,
workspace-specific fact during this interactive run — a naming/tagging
convention, a custom property worth weighing, a partner working preference, the
kind of thing `saved-context.md` describes — offer to persist it, and write the
bullet only on their OK. Never in Routine/non-interactive mode (§10). Structured
config (timezone, DB mappings, member IDs) is **not** context — it belongs in
`config.json`, so keep it there rather than in a context bullet.

### 3. Scaffold or adopt the internal databases (Inbox, Journal, Archive)

Under the root, scaffold or adopt Inbox, Journal, and Archive using the
schemas in `schema.md` and its internal-DB adopt/patch rules. These three are
plugin-owned and stay canonical — unlike task DBs (§3b, §7), they are safe to
additively patch.

- For each database, `notion-search`/`notion-fetch` under the root for a
  child matching the display-name convention (`Inbox`, `Journal`, `Archive` —
  tolerating a legacy leading emoji in the title).
- **Missing:** create it with `notion-create-database` using a **plain title**
  (`Inbox`/`Journal`/`Archive`) and the exact properties and types from
  `schema.md`. `notion-create-database` has no icon field, so immediately
  **set the native icon** with a follow-up `notion-update-page` on the new
  database's page id (`⚡`/`📓`/`🗑`; use `command: "update_properties"`,
  `properties: {}`, `icon: "…"` — per the Icons convention).
- **Existing:** verify every required property exists with the right type;
  **add** any missing properties with `notion-update-data-source` (DDL).
  Never drop or retype an existing property. Also **repair icon/title** if
  needed (set the native icon, strip any leading emoji from the title).
- **Type conflict** (a required property exists with the wrong type): do
  **not** mutate it. Report the conflict for that database and skip it —
  continue with the remaining databases.

Record each database's `data_source_url` for `config.json`.

**Record the Inbox `Triage` values map.** Alongside the Inbox's
`data_source_url`, record `inbox.triage_values` (the role→value map defined in
`schema.md`) so `capture`/`triage` resolve the right `Triage` values:

- **Scaffolded Inbox** (setup just created it): record the default map
  `{ "new": "New", "processed": "Promoted", "kept": "Kept" }` — an identity map
  over the values setup created.
- **Adopted Inbox** (an existing DB, including a consolidated Raw DB): fetch the
  `Triage` select's options. If they are exactly `New`/`Kept`/`Promoted`
  (case-insensitive), record the default map above. Otherwise —
  **interactive:** echo the options and ask which fills `new` (the unprocessed
  queue value) and which fills `processed`; record `kept` only if the user
  names a third option; record the confirmed map. **Non-interactive (Routine):**
  record only the roles whose value matches an option by exact name (`New`→`new`,
  `Processed`/`Promoted`→`processed`, `Kept`→`kept`); if `new` or `processed`
  cannot be matched, **omit `inbox.triage_values` entirely** (skills then fall
  back to the `schema.md` default) and report it as a pending pick for a later
  interactive run, mirroring the wiki/Waiting pending-manual-step pattern (§10).
  Never guess an unmatched role.

### 3b. Discover or scaffold the personal task DB (adopt-first)

The personal task DB (`tasks_personal`) is **not** forced into the scaffold
shape. Follow `task-db-mapping.md`'s adopt-never-mutate contract:

If step 2 loaded a **confirmed** mapping for `tasks_personal`, re-verify its
mapped properties still exist (`notion-fetch` the schema) and skip the rest
of this section. If it loaded an **unconfirmed** mapping, resume
disambiguation on just the `unconfirmed_roles` rather than starting over.

Otherwise, discover from scratch:

1. **Search for candidates.** `notion-search` the workspace for databases
   that look like a task DB (a title property plus a status- or
   select-typed property; a display name like "Tasks"/"To-do" (with or without
   a leading emoji) is a strong signal but not required). Present the candidates found and
   ask the user which one (if any) is their personal task DB. In
   non-interactive mode: **zero candidates found** → proceed to the scaffold
   branch below (step 3); **exactly one obvious candidate** → adopt it (step
   2); **two or more ambiguous candidates** (more than one plausible match,
   none obviously "the one") → do **not** scaffold and do **not** guess —
   leave `tasks_personal` unresolved for this run, record the candidate list
   as a pending pick (mirroring how ambiguous *roles* are recorded under
   `unconfirmed_roles` below, and the wiki/Waiting "pending manual step"
   pattern in §10), and report it. Scaffolding a new DB only happens when no
   candidate exists at all.
2. **Adopt + map the chosen candidate** (no DDL, ever, against this DB):
   - Fetch its data source schema.
   - Run **role inference** (heuristics below) to identify `title`,
     `status`, and any of `due`/`scheduled`/`assignee`/`priority`/`project`/
     `source` present.
   - Derive `status_values` (`open_default`, `done[]`) from the status
     property's groups.
   - **Interactive:** echo the inferred mapping and `status_values` back to
     the user; let them correct any ambiguous role; once confirmed, record
     `"confirmed": true`.
   - **Non-interactive (Routine):** run inference, record the mapping with
     any ambiguous roles listed under `unconfirmed_roles`, and report them
     as pending picks — mirror the wiki/Waiting "pending manual step"
     pattern (§10). **Never guess** an ambiguous role.
   - If `title` or `status` cannot be identified at all (no candidate
     property), fail loudly for this DB and stop — per `task-db-mapping.md`'s
     fail-loudly rule.
3. **If the user has no existing personal task DB**, scaffold `Tasks`
   using the scaffold-default shape from `schema.md` (`notion-create-database`,
   plain title `Tasks`; then set the native icon `✅` via a follow-up
   `notion-update-page`, per the Icons convention)
   and give it an **identity mapping** — `properties` naming exactly the
   properties just created (`title:"Name"`, `status:"Status"`, `due:"Due"`,
   `assignee:"Assignee"`, `priority:"Priority"`, `project:"Project"`,
   `source:"Source"`) and `status_values` from Notion's default groups
   (`open_default:"Not started"`, `done:["Done"]`). An identity mapping is
   `confirmed: true` by construction — there's nothing to disambiguate in a
   shape setup just created.

Record `tasks_personal`'s `data_source_url` plus its `properties`/
`status_values`/`confirmed` (or `unconfirmed_roles`) for `config.json` (§9).

**Role-inference heuristics** (full contract and worked examples in
`task-db-mapping.md` — summarized here for execution):

- `title`-typed property → `title` (always unambiguous, exactly one exists).
- `status`-typed property → `status`. If none exists but a select property is
  named like "Status"/"State", it's a `status` candidate — ask which select
  is the status driver (Routine: mark `status` unconfirmed).
- Date properties → `due`/`scheduled`. One date property → `due`. Multiple →
  **ask which is the hard deadline (`due`) vs. the do/work date
  (`scheduled`)** (Routine: mark both unconfirmed).
- Person property → `assignee` (ask if multiple exist).
- Properties named/typed like "Priority"/"Project"/"Source" → `priority`/
  `project`/`source` by name+type match; omit the role entirely if no match.
- `status_values`: if `status` is Notion's native `status` type, use its
  built-in groups — To-do group → `open_default` (ask if more than one
  To-do value exists), Complete group → `done[]`. If `status` was filled by
  a plain select (no groups), ask the user which values are "done" and which
  is the open default. Always echo the derived `status_values` and confirm
  before writing (Routine: auto-infer, flag ambiguous grouping as
  `unconfirmed_roles`).

### 4. Wiki (guided, UI-only)

`notion-search`/`notion-fetch` under the root for a wiki. Notion wikis
cannot be created via MCP — "Turn into wiki" is UI-only (see
`notion-conventions.md`).

If no wiki is found, walk the user through it:

1. Create a page titled `Wiki` (plain title + `icon: 📚`) under the root (you
   may create this page via `notion-create-pages`, or ask the user to create it).
2. Ask the user to open it in Notion and choose "Turn into wiki" from the
   page menu.
3. Ask the user to paste the resulting page URL back.

Once you have the URL, adopt it: fetch it to confirm it's now a wiki, and
record `wiki.database_id` (the wiki page — the parent target other skills
use when creating new wiki pages) and `wiki.data_source_url`
(`collection://…`, used only for structured queries against the wiki) for
`config.json`. Remind the user (once, not per-run) that new wiki pages must
be parented to the wiki **page** URL, not its data source, and that custom
properties like Tags are human-only — MCP can only set Verification.

If a wiki is already found, adopt it directly (no manual step needed) and
skip to step 5.

In non-interactive/Routine mode (see §10), do not prompt for this step —
report it as a pending manual step instead.

### 5. Waiting status option (guided, UI-only — scaffolded personal DB only)

This step applies **only when setup scaffolded** the personal task DB in
§3b (step 3, no pre-existing DB adopted). Never add a status option to a DB
you don't own — an adopted DB (personal or shared) is handled by the
inference branch below instead.

For a **scaffolded** DB: check whether its Status property already has a
**Waiting** option by fetching the Tasks data source schema. If it's
missing, instruct the user to do a one-time manual UI step: open the Tasks
database in Notion, edit the Status property, and add a **Waiting** option
to the **In progress** group. MCP cannot add or rename status options —
Notion only allows creating the three defaults (Not started / In progress /
Done) via DDL (`notion-conventions.md`). After the user confirms, re-fetch
the Tasks data source schema, verify **Waiting** is now present, and record
it as `status_values.waiting`. Do not assume success from the user's word
alone — verify. In non-interactive/Routine mode, do not prompt; report this
as a pending manual step instead.

For an **adopted** personal or shared task DB: never add a status option.
Instead, **infer** a waiting/blocked value from the Status property's groups
during role inference (§3b/§7) — e.g. an In-progress-group value like
"Awaiting response" — and record it as `status_values.waiting`. If no such
value exists, leave `waiting` unmapped and let dependent behavior (e.g.
"blocked on partner" in `two-person-rules.md`) degrade gracefully.

### 6. AGENTS page + config block

The `AGENTS` page is the machine-facing control file — an AGENTS.md-style page
that carries the fenced config block (the discovery anchor) and doubles as the
workspace's Notion AI instructions (§6a). It is **not** the human dashboard;
that is a separate `Home` page (§6b).

Under the root, `notion-search`/`notion-fetch` for an `AGENTS` child page
(title `AGENTS`, a leading emoji tolerated):

- **Found:** adopt it. Repair its icon/title if needed (`icon: 🤖`, plain
  title `AGENTS`).
- **Not found, but a legacy `Home` page carrying a config block exists**
  (the pre-split layout): **migrate it in place** — rename that page's title
  to `AGENTS`, set `icon: 🤖`, and restructure its body to the AGENTS.md-style
  template below, preserving/updating the existing fenced config block. Do not
  create a second page.
- **Neither exists:** create a fresh `AGENTS` page (`notion-create-pages`,
  plain title + `icon: 🤖`) with the template body below.

Record `agents_page` for `config.json`.

Write the fenced ```json config block into AGENTS's body using `insert_content`
(concurrency-safe append/insert), never a whole-page replacement — this avoids
clobbering concurrent human edits per `notion-conventions.md`. The block's keys
and structure must mirror `config.json` exactly (§9). If a config block already
exists, update it in place rather than duplicating it.

The AGENTS body reads as instructions first, config last (so it works as a
Notion AI instructions page), e.g.:

```
# AGENTS

The AGENTS.md of this Notion second brain — instructions for how AI should work
here, and the config block the plugin's skills read. Humans: use the Home page.

## How to work in this workspace
- File captures in Inbox; promote to Tasks/Journal/Wiki during triage.
- Discard by moving rows to Archive (there is no page-trash).
- Keep answers concise and cite the source page when answering from the brain.

## Conventions (for the plugin's skills)
- Skills address databases by data_source_url from the config block, never by name.
- Queries are single-data-source only.

## Config
​```json
{ …mirror of config.json (§9)… }
​```
```

### 6a. Notion AI instructions (guided, UI-only — optional, opt-in)

After the AGENTS page exists, offer to make it the workspace's **Notion AI
Instructions** page so Notion AI follows it. This cannot be done via MCP — it
is UI-only (see `notion-conventions.md`).

If the user opts in, walk them through it: open the AGENTS page → `···` →
`Use with AI` → `Use as AI Instruction`. Warn that only **one** instructions
page is active at a time, so this replaces any current one. This is not
verifiable via MCP — accept the user's confirmation. Skipping is fine; the step
is additive and can be done on a later run.

In non-interactive/Routine mode (see §10), do not prompt — report it as a
pending manual step instead.

### 6b. Home page (human dashboard)

Under the root, `notion-search`/`notion-fetch` for a `Home` child page (title
`Home`, a leading emoji tolerated). Note: if a legacy `Home` was just migrated
to `AGENTS` in §6, none remains — create a fresh one.

- **Found:** adopt it; repair icon/title if needed (`icon: 🏠`, plain title
  `Home`). If it still holds a leftover config block, remove it — config lives
  on AGENTS now.
- **Not found:** create it (`notion-create-pages`, plain title + `icon: 🏠`).

Keep it **simple and human-facing — quick links only**, no config block and no
scripted views. Example body:

```
# Home

Your second brain.

- Inbox — where captures land
- Tasks — what to do
- Journal — daily notes
- Wiki — reference
- Archive — discarded items
```

Render the links as Notion page/DB mentions or child-page links. Record
`home_page` for `config.json`.

### 7. Shared spaces (zero or more)

Ask the user whether to register any shared spaces now (e.g. a space shared
with a partner). This step is optional and additive — zero spaces is a
valid outcome, and re-running `setup` later can register more.

For each named shared space:

1. Locate its task database (search under the space's root, or ask the user
   for it) — the display name and shape are whatever already exists there;
   don't expect the scaffolded `Tasks` shape.
2. **Discover and map it — never mutate.** If step 2 already loaded a
   confirmed mapping for this space's task DB, re-verify its properties
   still exist and skip to step 3. Otherwise run the same role-inference and
   `status_values`-derivation heuristics as §3b against this DB's schema:
   identify `title`/`status` (no candidate property at all for either → fail
   loudly, per §3b's rule; a candidate found but ambiguous → `unconfirmed_roles`,
   not a failure) and any optional roles present, derive `status_values` from its Status
   groups, echo and confirm the mapping (interactive), or record ambiguous
   roles under `unconfirmed_roles` and report them as pending picks
   (non-interactive/Routine — never guess). This DB is never scaffolded and
   never receives DDL — no `notion-update-data-source` call against it,
   ever, regardless of what properties it's missing relative to the
   scaffold shape.
3. Populate `members` by resolving each named member via `notion-get-users`.
4. Append the space to `shared_spaces` in `config.json`: `name`, `root`,
   `members`, and `tasks` (`data_source_url` plus the `properties`/
   `status_values` mapping — including `waiting` when one was inferred —
   and the `confirmed`/`unconfirmed_roles` marker), matching the exact
   shape in `schema.md`'s config example.

In non-interactive/Routine mode, skip prompting and leave `shared_spaces` as
whatever is already in the adopted `config.json` (do not invent spaces).

### 8. Query-gating test

Issue one single-source filtered+sorted `notion-query-data-sources` query
against a scratch data source (or the Inbox data source if no scratch DB is
available) — never a cross-data-source query, per
`notion-conventions.md`.

- **Succeeds:** record the query path as `structured` (the workspace is on
  Business + Notion AI or higher — a single-source query is ungated there).
- **Throws the plan-gate error** (the `400` signature in
  `query-plan-gating.md` — a single-source query throwing it means a
  Plus/Free tier): fall back to scoped `notion-search` + `notion-fetch`,
  record the query path as `fallback`, and surface the
  upgrade-vs-permanent-fallback decision to the user (don't silently pick
  one). A generic 400 from a malformed probe filter is a bug in the test, not
  a plan gate — fix the probe rather than recording `fallback`.

Store the observed path in the run report — it's advisory for other
query-dependent skills, not written into `config.json`'s schema.

### 9. Write config.json

Write `config.json` — on a bootstrap run to the **Step 0 repo's absolute path**
(`<location>/config.json`), on an adopt run (Step 0 skipped) to the
launch-folder cwd as before — with every key defined in
`schema.md`: `user`, `notion_user_id`, `second_brain_root`, `home_page`,
`agents_page`, `wiki` (`database_id`, `data_source_url`),
`inbox` (`data_source_url`, plus `triage_values` when recorded in §3),
`tasks_personal`, `journal`, `archive` (each with `data_source_url`, except
`tasks_personal` when unresolved — see below), `shared_spaces` (array,
possibly empty), and `preferences` (`timezone`, `calendar_tool`). **Keep
`timezone` in the key set** — it must not be dropped. Use the exact key
names from `schema.md` — do not add, rename, or drop keys. `tasks_personal`
and every `shared_spaces[].tasks` now also carry `properties` +
`status_values` (plus `confirmed`/`unconfirmed_roles`) alongside
`data_source_url`, per §3b/§7 and the exact shape in
`schema.md`/`task-db-mapping.md` — except when §3b left `tasks_personal`
unresolved (2+ ambiguous candidates, non-interactive), in which case write
`tasks_personal` as `{"pending_selection": true, "candidates": [...]}` with
no `data_source_url`, mirroring how `unconfirmed_roles` marks a mapping
incomplete.

**Derive and persist `preferences.timezone`.** After `notion-get-users`
(§1), if the loaded/adopted config has no `preferences.timezone` set:
interactively, ask the user for their IANA timezone (e.g. `America/New_York`)
and write it. In Routine/non-interactive mode, keep whatever timezone the
adopted config already has — **never** fall back to the running session's
clock, which is UTC in cloud/Routine/Cowork sessions (`task-db-mapping.md`).
If there is no adopted config and no way to prompt, leave `timezone` unset
and report it as a pending manual step (§10) rather than guessing one.

Mirror the identical JSON into the AGENTS page's fenced config block (§6, via
`insert_content`).

`config.json` **is committed** in the user's *private* second-brain repo —
that is how remote sessions read it. On an **adopt** run the auto-commit Stop
hook (Step 0.3) picks it up automatically at session end. On a **bootstrap**
run, commit and push it immediately instead of waiting on the hook — this
run's own Stop hook can't fire against the new repo yet (see §0.6):

```bash
git -C "<location>" add -A
git -C "<location>" commit -m "chore: add config.json"
git -C "<location>" push
```

It is still **never** committed to the public **plugin** repo, and this skill
still never writes personal data (names, IDs, workspace names) into the
plugin's own output files.

### 10. Idempotency + Routine mode

Re-running `setup` on an already-populated folder/workspace must change
nothing that's already present and correct — it re-verifies each step,
finds everything present, and reports "all present": still additively
patching internal DBs per step 3's rules where properties are missing, and,
for task DBs, reading the stored mapping (§2) rather than re-discovering or
mutating anything (§3b, §7).

If invoked non-interactively (Routine mode, no way to prompt for the wiki
URL, the Waiting-option confirmation, the AGENTS→AI-instructions step, or
task-DB mapping disambiguation): do not block waiting for input. Instead,
produce a report listing exactly what setup *would* create or still needs from
the user (e.g. "wiki not yet turned into a wiki — visit AGENTS for
instructions", "Waiting status option still missing on Tasks", "AGENTS not yet
set as the Notion AI instructions page", "task DB mapping has unconfirmed
roles: due, priority — pending pick"), and complete the rest of the run
(databases, AGENTS, Home, config.json, gating test) as far as it can go
non-interactively.

## Smoke test

Smoke test (run against a live Notion workspace):
1. From an empty working directory (no config.json, plugin not yet enabled
   here), run setup. Expect Step 0 to bootstrap: create a private GitHub repo
   under `~/src/<firstname>-notion-second-brain` (or `~/Documents/…`), inject
   `.gitignore` (not ignoring config.json), `.claude/settings.json` (marketplace
   + enabledPlugins + a Stop autocommit hook), `.claude/hooks/autocommit.sh`
   (executable), `CLAUDE.md` (with an empty `## Second brain context` section and
   the `ns2b:context` marker pair), and a personal `README.md`, then push a first
   commit. Re-running inside that repo skips Step 0 (0.0 detection).
2. Expect setup to: ping the connector (notion-get-users) OK; find or create the
   "Second Brain" root; create/adopt Inbox, Journal, Archive with the
   schema.md schemas — each with a plain title and a native icon (no emoji in
   the title); discover-or-scaffold the personal task DB per §3b; create/adopt
   the AGENTS page with a fenced config block and a separate simple Home page;
   write config.json with all keys populated.
3. Expect setup to run the gating test: a single-source filtered+sorted query
   against a scratch DB, and report path = "structured" or "fallback".
4. Expect setup to surface the manual UI steps (create `Wiki` → "Turn into
   wiki" → paste URL back; add "Waiting" status option — only if the personal
   DB was scaffolded; optionally set AGENTS as the Notion AI instructions
   page), then verify each after the user confirms.
5. Re-run setup on the now-populated folder → it changes nothing (idempotent) and
   reports "all present".
Assertions: config.json exists with non-empty second_brain_root, inbox
(`data_source_url`, plus `triage_values` when setup recorded it per §3),
tasks_personal, journal, archive, `home_page`, and `agents_page`; Notion shows
the four DBs with correct properties (task DB unchanged from before the run —
no DDL against it) and every page/DB shows exactly one icon (native icon, plain
title); the AGENTS page contains the config block and the Home page does not;
`tasks_personal`'s mapping is recorded with `properties`/`status_values` and a
`confirmed` marker — identity-mapped if scaffolded, discovered if adopted; a
re-run with no local `config.json` reads the mapping from the AGENTS block and
skips re-disambiguation; a non-interactive run records `unconfirmed_roles` for
any ambiguous role and reports it as a pending pick instead of guessing.
On a bootstrap run, the new repo exists with an origin remote, contains the
five injected files, and its config.json is committed (not gitignored); a
second run started with `cd <repo> && claude` detects config.json and skips
Step 0. The bootstrapped `CLAUDE.md` carries a `## Second brain context` section
with the `ns2b:context` marker pair; a re-run against a repo whose `CLAUDE.md`
is missing that section recreates it, and one with a duplicated section collapses
it to a single section preserving every bullet (§2).

## Errors

- **Connector unavailable/unauthenticated** (`notion-get-users`, or any
  other Notion MCP call, fails): stop immediately, state the failure
  plainly, and tell the user to check `/mcp` and confirm they're logged in
  with their claude.ai account. Never fake success or continue as if the
  workspace were reachable.
- **Multiple root candidates:** stop and ask the user which `Second Brain`
  page to use. Never guess.
- **Property type conflict on an existing internal DB (Inbox/Journal/
  Archive):** never mutate it — report the conflicting database and
  property, skip that database, and continue with the rest of the run. Task
  DBs never receive DDL, so a "wrong type" on a task DB isn't a
  mutation-blocked case — see the next error.
- **Un-identifiable `title`/`status` on a task DB** (personal or shared):
  report and skip that DB — never mutate it, and never guess a role. Record
  it as unmapped rather than fabricating a mapping.
- **Rate limiting (`429`):** honor `Retry-After` and back off before
  retrying, per `notion-conventions.md`.
- **Large/async writes:** if a write returns an async task, poll
  `notion-get-async-task` rather than assuming completion.
