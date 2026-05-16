# Project: spec skill (OpenSpec lifecycle wrapper)

## Purpose

`spec` is a Claude Code skill that wraps [Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec) with lifecycle gates the upstream tool lacks. The wrapper itself is self-hosted in OpenSpec format here so that future changes to the wrapper's behavior (adding modes, tuning thresholds, adding hooks, changing retreat rules) go through the `/opsx:propose` → `/opsx:apply` → `/opsx:verify` → `/opsx:archive` pipeline with captured rationale.

**Why wrap OpenSpec instead of using it directly:**
1. OpenSpec has no gate — assumes the user already decided to spec-drive, which mis-fires on RA / 國考 / 教學 / meta-work projects
2. OpenSpec has no retro — mature projects with git tag + release history waste effort if propose-from-zero is the only path
3. OpenSpec has no lifecycle integration — no auto context warm-up on resume, no decision capture below `/opsx:propose` granularity, no session-end validate reminder, no auto-compact defense

**What this spec skill adds (three capabilities):**
1. `assessment-and-bootstrap` — rubric gate + OpenSpec-safe init + git-history retro
2. `session-continuity` — resume warm-up + `/spec note` + handoff + clear shortcut (cross-session working memory via file system)
3. `compact-defense` — 3-layer automation (proactive warn + passive checkpoint + manual handoff) with meta-work exception for editing `~/.claude/` itself

## Stack & Constraints

- **Skill file**: `~/.claude/skills/spec/SKILL.md` (34K markdown)
- **Hook scripts**: `~/.claude/scripts/spec-*.sh` (four scripts: session-start, context-watch, pre-compact, plan-mode)
- **OpenSpec CLI**: `npm install -g @fission-ai/openspec@latest` (requires Node ≥ 20.19; current: 1.3.0)
- **Integration points**: `~/.claude/settings.json` hooks registry + `~/.claude/CLAUDE.md` Skill Router section
- **Symlink awareness**: `~/.claude/{CLAUDE.md,imports,scripts,skills}` may be symlinked to an external dotfiles repo (e.g. `~/dotfiles/claude-config/`) for version control — all `find` operations on `~/.claude/` MUST use `-L` to follow symlinks
- **Self-hosting location**: this `openspec/` lives at `~/.claude/skills/spec/openspec/` (skill-local, not global `~/.claude/openspec/`), because this spec skill is a single self-contained cluster and other skills do not depend on its contract

## Out of scope

- Re-implementing OpenSpec primitives (propose / apply / archive / verify) — those always go through `/opsx:*`
- Replacing `research_plan.json` (statistics workflow) or MA directory structure (`01_protocol/` + `09_qa/`) — spec retreats in those contexts
- Auto-committing any file — curator mode: every non-trivial write needs user confirmation
- Auto-invoking `/clear` — Claude Code security limit, `/clear` must be user-typed

## Migration history

See `~/.claude/skills/spec/MIGRATION.md` for the path from `spec-driven` (single-file spec.md curator, 2026-04-18) → `spec` (OpenSpec wrapper, 2026-04-21).
