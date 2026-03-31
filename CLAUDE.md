# CLAUDE.md - AI Assistant Rules for This Repository

## Repository Purpose

This is a **study-only** repository for learning and analyzing the Claude Code source code.
All work here is reading, analyzing, and producing study reports — never modifying the original source.

## Iron Rule: src/ is READ-ONLY

**NEVER modify, edit, write, or delete any file under `src/`.**

This rule has NO exceptions. The source code must remain exactly as-is to serve as a faithful reference.

- Do NOT fix bugs, typos, formatting, or lint issues in `src/`
- Do NOT add comments, annotations, or type hints to files in `src/`
- Do NOT refactor, rename, or reorganize anything in `src/`
- Do NOT create new files inside `src/`

If a study task seems to require modifying source code, you are misunderstanding the task. Ask for clarification.

## What You CAN Do

- **Read** any file in the repository (including `src/`)
- **Create** study reports, notes, and analysis documents in the project root or study directories
- **Create** diagrams, summaries, and mapping documents
- **Run** git commands for committing study outputs
- **Search** the codebase with Grep, Glob, and other read-only tools

## Study Output Conventions

- Study reports go in the project root (e.g., `S01-Production-Mapping-Report.md`)
- Use descriptive filenames that indicate the topic
- Reports should map theoretical concepts to actual source code locations (file:line)

## Commit Style

- Commit messages should describe what was studied/analyzed, not "what changed"
- Example: `Add S01 agent loop mapping report`
- Example: `Add tool execution architecture analysis`
