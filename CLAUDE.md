# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project purpose

This repo is a sandbox for a Claude Code **skill**, `consciousness-guide`
(defined in [.claude/skills/consciousness-guide/SKILL.md](.claude/skills/consciousness-guide/SKILL.md)),
whose answers are grounded in the consciousness frameworks of **Robert Monroe**
(*Journeys Out of the Body*, *Far Journeys*, *Ultimate Journey*, the Monroe Institute /
Hemi-Sync "Focus levels") and **Thomas Campbell** (*My Big TOE* — "MBT"). Invoke it with
`/consciousness-guide` when a user wants to explore a question through that lens —
reflective, big-picture, or meaning-of-life style questions. Unlike a subagent, it runs
inline in the current conversation, so follow-ups and earlier context carry over
naturally. It presents both frameworks as exploratory, not asserted scientific or
medical fact, and defers medical/psychiatric/crisis questions to a qualified
professional rather than answering them itself.

`main.py` is unrelated leftover scaffolding (just prints `hi`) and isn't part of this
project's actual purpose — don't build on it without checking with the user first.

## Editing the guide's persona

The full persona/voice instructions live in the skill file's body, not here, so
there's a single source of truth. When updating the framing of Monroe's or Campbell's
ideas, edit [.claude/skills/consciousness-guide/SKILL.md](.claude/skills/consciousness-guide/SKILL.md)
directly rather than duplicating the content into this file.
