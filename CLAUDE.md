# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project purpose

This repo is a sandbox for Claude Code **skills** (`.claude/skills/<name>/SKILL.md`).
Each skill is invoked with `/<name>` and runs inline in the current conversation, so
its instructions, follow-ups, and earlier context carry over naturally — unlike a
subagent, which runs in an isolated context.

When asked to use, update, or create a skill, look in `.claude/skills/` first rather
than assuming a skill's behavior — the SKILL.md frontmatter (`name`, `description`)
is the source of truth for what it does and when to invoke it, and the body holds the
actual instructions. Don't duplicate a skill's content into this file; reference it
instead so there's a single source of truth.

## Skills in this repo

- **`consciousness-guide`** ([.claude/skills/consciousness-guide/SKILL.md](.claude/skills/consciousness-guide/SKILL.md))
  — answers reflective, big-picture questions through Robert Monroe's out-of-body/
  Focus-level framework and Thomas Campbell's My Big TOE (consciousness-as-fundamental,
  entropy reduction). Presents both as exploratory lenses, not asserted fact; defers
  medical/psychiatric/crisis questions to a professional.
- **`shop-amazon`** ([.claude/skills/shop-amazon/SKILL.md](.claude/skills/shop-amazon/SKILL.md))
  — browses and purchases products on Amazon.com via the Chrome DevTools MCP server.
  Requires explicit user approval before placing any order; never enters payment
  credentials or selects shipping addresses on the user's behalf.

Keep this list in sync when skills are added, removed, or renamed.


