# Notion Second Brain

A Claude Code plugin that runs a two-person second brain on Notion through the
built-in Notion connector. Capture, daily brief, inbox triage, and cited query —
across Claude Code, Cowork, claude.ai chat/mobile, cloud sessions, and Routines.

## Requirements

- The **Notion connector** connected once at claude.ai (Settings → Connectors).
  All skills use it as their only transport.
- A Notion workspace on a paid plan (two members exceed the free block cap).

## Install (Claude Code)

```
/plugin marketplace add brandonpeebles/notion-second-brain
/plugin install notion-second-brain@notion-second-brain-marketplace
```

Then run `/notion-second-brain:setup` from your launch folder.

<!-- Full install matrix, Routine recipes, backup guidance: see Task 9. -->
