# Notion Second Brain

A Claude Code plugin that runs a two-person second brain on Notion through the
built-in Notion connector. Capture, daily brief, inbox triage, and cited query —
across Claude Code, Cowork, claude.ai chat/mobile, cloud sessions, and Routines.

## What it is

Five skills, all talking to your Notion workspace exclusively through the
built-in Notion connector (no MCP config files, no hooks to wire up):

- **setup** — discovers or scaffolds the workspace structure, writes
  `config.json`, verifies the Notion connector, and registers shared spaces.
  Idempotent and safe to re-run.
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

## Requirements

- The **Notion connector** connected once at claude.ai (Settings →
  Connectors). All skills use it as their only transport.
- A Notion workspace on a **paid plan** — two members exceed the free plan's
  block cap.

## Install

### Claude Code

```
/plugin marketplace add brandonpeebles/notion-second-brain
/plugin install notion-second-brain@notion-second-brain-marketplace
```

If the repo is private, this works once you've added Claude's GitHub
connection at claude.ai (Settings → Connectors). If you can't add that
connection, the documented fallback is uploading the skills as a `.zip`.

### claude.ai chat/mobile

Settings → Customize → Plugins → "Add from a repository", then point it at
`brandonpeebles/notion-second-brain`.

### Cowork, cloud sessions, and Routines

Point them at the same repository — the Notion connector supplies the
transport, so no extra setup is needed beyond what's above.

## First run

Create a launch folder for the plugin's local state, e.g. `~/second-brain/`,
and run `setup` from inside it:

```
/notion-second-brain:setup
```

`setup` writes `config.json` into that folder. The file holds your
workspace's database and page IDs and is gitignored — it never gets
committed.

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

The repo itself ships no personal data. Every workspace-specific
identifier — database IDs, page IDs, your user ID — lives only in your
local `config.json` and the Home page's config block inside your own
Notion workspace, never in this repository.
