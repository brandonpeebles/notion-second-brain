# Notion Second Brain

A Claude Code plugin that runs your second brain on Notion through the
built-in Notion connector — personal, with optional spaces you share with
others. Capture, daily brief, inbox triage, and cited query. Verified on
the Claude Code CLI with the claude.ai Notion connector; Cowork, claude.ai
chat/mobile, cloud sessions, and Routines are supported in principle but
unverified end-to-end (see Install below).

## What it is

Five skills, all talking to your Notion workspace exclusively through the
built-in Notion connector (no MCP config files, no hooks to wire up):

- **setup** — discovers or scaffolds the workspace structure, writes
  `config.json`, verifies the Notion connector, and registers any shared spaces.
  Adapts to your existing task databases (maps to their schema, never
  modifies them). Idempotent and safe to re-run.
- **capture** — zero-decision capture of a task, note, idea, or reference;
  routes to the Inbox, or straight to Tasks when unambiguous. Phone-first,
  echoes parsed dates back for confirmation.
- **today** — the daily brief: overdue and due-today tasks across all task
  databases, today's calendar events, and an upserted journal entry. Runs
  unattended in Routines (reports only, never prompts).
- **triage** — sweeps the Inbox, proposing each item be promoted to a task,
  wiki page, or journal entry, or archived — then executes the batch after
  one confirmation.
- **query** — answers questions from the second brain with structured
  filters or scoped semantic search over the wiki, tasks, and journal,
  returning a cited answer.

### Bring your own task database

`setup` discovers your existing task database(s) rather than forcing a
fixed schema on you: it maps each one's real property names and status
values (title, status, due date, assignee, priority, ...) into
`config.json`, and every other skill reads and writes through that mapping.
Your database keeps its own shape — nothing is added, renamed, or removed.
If you don't have one yet, `setup` scaffolds a default `Tasks` database
instead.

## Requirements

- The **Notion connector** connected once at claude.ai (Settings →
  Connectors). All skills use it as their only transport.
- Connectors sync into the Claude Code CLI only when your claude.ai
  subscription is the active auth method — they do **not** load under
  `ANTHROPIC_API_KEY`/`ANTHROPIC_AUTH_TOKEN`/`apiKeyHelper`/Bedrock/Vertex.
  Log into Claude Code with your claude.ai account (`/login`); the Notion
  connector does not load under an API key.
- A Notion workspace with **structured queries** available: single-source
  structured queries (server-side filter/sort) work on Plus (verified on
  this workspace). Cross-source queries in one call require Enterprise; the
  plugin avoids them by design — it queries each task database separately
  and merges the results — so cross-source gating is not a limitation in
  practice. Each query-touching skill auto-detects a plan-gate on a given
  query and falls back to scoped search+fetch when that query is gated. (The
  exact gate on the single-source tool — a Notion-AI add-on vs. plan tier —
  is unconfirmed; this describes observed behavior, not a guaranteed tier
  requirement.) See `skills/shared-references/query-plan-gating.md` for the
  full mechanism.

## Install

### Claude Code

```
/plugin marketplace add brandonpeebles/notion-second-brain
/plugin install notion-second-brain@notion-second-brain-marketplace
```

If the repo is private, this works once your **local git auth** can reach
it — `gh auth login`, or an SSH key added to your GitHub account. Set
`GITHUB_TOKEN` too if you want background auto-updates of the marketplace to
work. There is no install-from-local-clone fallback; the `/plugin
marketplace add` command needs to fetch the repo itself.

### claude.ai chat/mobile

Settings → Customize → Plugins → "Add from a repository", then point it at
`brandonpeebles/notion-second-brain`. **Unverified** — this is a real
claude.ai install mechanism, but end-to-end plugin install and namespaced
skill invocation on claude.ai chat/mobile has not been tested against this
plugin.

### Cowork, cloud sessions, and Routines

**Unverified** — plugin install and skill invocation on Cowork, cloud
sessions, and Routines have not been tested end-to-end. The Notion connector
itself supplies the transport wherever it's connected, but this README's
tested and confirmed surface is the **Claude Code CLI with the claude.ai
Notion connector**; treat anything beyond that as unconfirmed until verified.

## First run

Create a launch folder for the plugin's local state, e.g. `~/second-brain/`,
then **`cd` into it before starting Claude Code** — a skill sees the
process's working directory, not an abstract "launch folder," so `setup`
only finds/writes `config.json` there if you actually `cd` first:

```
cd ~/second-brain/ && claude
```

Then run `setup`:

```
/notion-second-brain:setup
```

`setup` writes `config.json` into that folder. The file holds your
workspace's database and page IDs.

## Routine recipes

Not shipped as part of the plugin, but easy to wire up yourself once
`setup` has run:

- **Daily** — run `today` each morning for the brief.
- **Sunday** — run `triage` to sweep the week's Inbox. (A dedicated `tidy`
  skill for deeper cleanup is planned for a later release.)

## Backup

Notion is now the primary store for this data, with no git safety net
behind it. Schedule a periodic workspace export (manually, or via a
Routine) so you have an offline backup.

## Privacy

The repo itself ships no personal data — every workspace-specific
identifier (database IDs, page IDs, your user ID) lives only in `config.json`
and the AGENTS page's config block inside your own Notion workspace, never in
this repository.

`config.json` is written to **your launch-folder cwd** (see First run above)
— it is not part of this plugin's repo, so this plugin's own `.gitignore`
cannot protect it. If your launch folder happens to be inside a git repo you
track, add your own `.gitignore` entry for `config.json` there, or use a
launch folder that isn't a tracked repo at all.

For Cowork/cloud sessions, there is no durable local filesystem: the AGENTS
page's config block (inside your own Notion workspace) is the durable copy,
and any `config.json` on that surface is just a per-session cache — the
gitignore framing above doesn't apply there.

## Future work

The Notion **Public API** is ungated on Plus for structured queries, while
the hosted **MCP** query tool is plan-gated on Plus (see
`skills/shared-references/query-plan-gating.md`). A small Claude-Code-only
wrapper (a local script using a Notion integration token against the raw
Public API) could restore server-side filter/sort on Plus for `today` and
`query`. This would be Claude-Code-only — it needs a local script host and a
stored token, so it wouldn't help Cowork/chat/mobile — and if built, it
should be an optional backend the skills prefer when present and fall back
from when absent, not a replacement for the existing MCP + search/fetch
dual-path. Not implemented; noted here as a possible follow-up only.
