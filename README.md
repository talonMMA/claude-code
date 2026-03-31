# Claude Code Source - Study & Research

> This repository is a **personal study archive** of the Claude Code source code. Its sole purpose is to **read, analyze, and learn** the architecture and implementation patterns of a production-grade AI coding agent.

---

## Purpose

This repository is maintained for:

- Deep study of agentic loop architecture (Agent Loop, Tool Use, Context Management)
- Understanding production-grade TypeScript/React CLI system design
- Learning how LLM-powered coding agents work at the source level
- Writing analysis reports and study notes

**This is NOT a development fork. No modifications are made to the original source code.**

---

## Iron Rule

> **`src/` is READ-ONLY. Source code must never be modified.**
>
> All study outputs (reports, notes, diagrams) go in the project root or dedicated study directories.
> The original source must remain intact as a faithful reference.

---

## Study Outputs

| File | Description |
|------|-------------|
| `class-101-prototype-to-production/S01-Production-Mapping-Report.md` | Agent Loop: mapping S01 theory to production implementation |

---

## Repository Scope

Claude Code is Anthropic's CLI for interacting with Claude from the terminal to perform software engineering tasks such as editing files, running commands, searching codebases, and coordinating workflows.

- **Language**: TypeScript
- **Runtime**: Bun
- **Terminal UI**: React + [Ink](https://github.com/vadimdemedes/ink)
- **Scale**: ~1,900 files, 512,000+ lines of code

---

## Directory Structure

```text
src/                          # [READ-ONLY] Original source code
├── main.tsx                  # Entrypoint (Commander.js CLI)
├── query.ts                  # Core agent loop
├── tools/                    # 40+ tool implementations
├── services/                 # API, MCP, OAuth, analytics, etc.
├── screens/                  # REPL, Doctor, Resume
├── commands/                 # ~50 slash commands
├── components/               # Ink UI components
├── hooks/                    # React hooks
├── utils/                    # Utilities
├── bridge/                   # IDE integration
├── coordinator/              # Multi-agent orchestration
└── ...

class-101-prototype-to-production/  # Study reports series
  S01-Production-Mapping-Report.md
CLAUDE.md                          # AI assistant rules for this repo
```

---

## Ownership Disclaimer

- The original Claude Code source is the property of **Anthropic**.
- This repository is **not affiliated with, endorsed by, or maintained by Anthropic**.
- This archive exists solely for personal educational study.
