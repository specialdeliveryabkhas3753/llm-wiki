# Project: llm-wiki

## Context

llm-wiki implements Andrej Karpathy's "LLM Wiki" concept — a persistent, structured
knowledge base maintained by an LLM (Claude Code) using a dual-layer cache architecture
inspired by CPU memory hierarchies.

- **Owner:** Mehmet Goekce / MEMOTECH
- **License:** MIT (open-source)
- **Repository:** github.com/MehmetGoekce/llm-wiki
- **Status:** Initial release (v1.0), actively developed

## Architecture

- **L1 (Fast, auto-loaded):** Claude Code memory directory (~10-20 files). Rules, gotchas,
  credentials, identity. Loaded every session. Git-excluded.
- **L2 (On-demand):** Logseq or Obsidian wiki (~50-200 pages). Projects, workflows,
  research, deep knowledge. Queried via `/wiki` commands. Git-tracked.

## Tech Stack

- **Runtime:** Claude Code (CLI)
- **Dependencies:** bash, python3, git (no npm, no pip)
- **Wiki tools:** Logseq (outliner) or Obsidian (flat markdown)
- **Config:** `llm-wiki.yml` (YAML)
- **Format:** Markdown with tool-specific conventions

## Stakeholders

| Role | Who | Responsibility |
|------|-----|---------------|
| Maintainer | Mehmet Goekce | Architecture, releases, specs |
| Contributors | Open-source community | Features, bug fixes, templates |
| Users | Claude Code users | Install, configure, use /wiki commands |

## Constraints

- Zero external dependencies beyond bash, python3, git
- Must work on macOS, Linux, WSL
- Must support both Logseq and Obsidian (identical capabilities, different format)
- Config always reads from `llm-wiki.yml`
- Max 3 wiki pages loaded simultaneously (LLM context budget)
- Credentials MUST stay in L1 (L2 is git-tracked)
- Append-only updates (never overwrite existing wiki content)

## Specs

| Spec | Covers | Requirements | Scenarios |
|------|--------|-------------|-----------|
| specs/ingest.md | /wiki ingest — 5-phase source processing pipeline | 25 | 10 |
| specs/query.md | /wiki query — search, synthesis, write-back | 16 | 10 |
| specs/lint.md | /wiki lint — 9 automated health checks with auto-fix | 23 | 10 |
| specs/schema.md | Page types, properties, validation, format rules | 30 | 10 |
| specs/config.md | llm-wiki.yml loading, validation, error handling | 18 | 8 |
| specs/setup.md | setup.sh interactive installer (11 steps) | 28 | 10 |
| specs/l1-l2-routing.md | L1/L2 boundary decision logic | 22 | 8 |
| **Total** | **Complete system coverage** | **162** | **66** |
