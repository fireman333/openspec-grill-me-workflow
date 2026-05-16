# Spec: compact-defense

## Purpose

Claude Code's auto-compact compresses conversation state when context approaches the limit, which can drop in-flight decisions that were not yet persisted to disk. This capability defends against that loss through three complementary layers:

1. **Proactive warn** (UserPromptSubmit hook) — warn the model/user before compact fires, while there is still budget to run `/spec note` or `/spec handoff`
2. **Passive checkpoint** (PreCompact hook) — when compact is about to fire, auto-snapshot validate + in-progress state to disk so the next session can reconstruct
3. **Manual handoff** — user-driven `/spec handoff` at natural session ends

The three layers are complementary: layer 1 prevents, layer 2 backstops, layer 3 closes. A meta-work exception makes layers 1 + 2 fire even when CWD is in a skip-list, as long as the session has recently edited `~/.claude/` itself.

## Requirements

### Requirement: Proactive context warning tiers

The `spec-context-watch.sh` UserPromptSubmit hook SHALL monitor the current session's transcript jsonl size as a proxy for context usage and SHALL inject tiered reminders when thresholds are crossed.

Thresholds SHALL be configurable via environment variables `CONTEXT_WATCH_WARN_BYTES` and `CONTEXT_WATCH_STRONG_BYTES`, with defaults of 1_000_000 bytes (warn, ~35% context) and 1_600_000 bytes (strong, ~55% context). The strong tier SHALL leave at least 25–30% runway before Claude Code's auto-compact fires, giving the model budget to run a full `/spec handoff`.

#### Scenario: warn tier fires once per session

- **WHEN** transcript jsonl size first crosses `CONTEXT_WATCH_WARN_BYTES`
- **THEN** hook SHALL inject a reminder suggesting `/spec note "<decision>"` for any uncaptured decisions
- **AND** SHALL record the fire in `~/.claude/logs/context-watch-<session_id>.state` so it does not re-fire in the same session

#### Scenario: strong tier recommends handoff

- **WHEN** transcript jsonl size first crosses `CONTEXT_WATCH_STRONG_BYTES`
- **THEN** hook SHALL inject a stronger reminder recommending `/spec handoff` + manual `/compact "focus on <change>"`
- **AND** SHALL respect throttle (once per session per tier)

#### Scenario: env override changes thresholds

- **WHEN** `CONTEXT_WATCH_WARN_BYTES=500000` is set in `~/.claude/settings.json` env block
- **THEN** warn tier SHALL fire at 500_000 bytes instead of the default 1_000_000

### Requirement: Passive pre-compact checkpoint

The `spec-pre-compact.sh` PreCompact hook SHALL run `openspec validate --all` and dump in-progress change state to disk in the background when auto-compact is about to fire on an OpenSpec project. The hook MUST NOT block compact execution.

#### Scenario: snapshot written to openspec/decisions/

- **WHEN** auto-compact fires in a CWD with `openspec/` present
- **THEN** hook SHALL write `openspec/decisions/<YYYY-MM-DD_HHMMSS>-precompact-snapshot.md` containing in-progress changes list + validate summary + recovery hint
- **AND** validate log SHALL be written to `~/.claude/logs/openspec-precompact-<slug>-<ts>.txt`

#### Scenario: snapshot is readable by next resume

- **WHEN** next session runs `/spec resume`
- **THEN** the resume warm-up SHALL include the most recent pre-compact snapshot as part of its `openspec/decisions/` read

#### Scenario: hook must not block compact

- **WHEN** pre-compact hook fires
- **THEN** the validate + snapshot work SHALL run in a backgrounded subshell
- **AND** hook SHALL exit 0 regardless of validate outcome so Claude Code's compact is not blocked

#### Scenario: PreCompact output field constraint

- **WHEN** pre-compact hook injects a reminder message
- **THEN** hook SHALL use the top-level `systemMessage` field (not `hookSpecificOutput.additionalContext`)
- **BECAUSE** `additionalContext` is only valid for PreToolUse / UserPromptSubmit / PostToolUse — by the time PreCompact fires there is no "next prompt" to inject into

### Requirement: Meta-work exception

When CWD is in the default skip-list (`~/Desktop`, `~/Downloads`, `~/`, `~/.claude/*`) but the current session has edited `~/.claude/` itself recently, the warn/strong hook and pre-compact hook SHALL still fire. Editing the Claude configuration itself is a legitimate workflow that would otherwise be left undefended.

Meta-work detection SHALL use `find -L "$HOME/.claude" -type f -mmin -30` with symlink-following. The `-L` flag is CRITICAL because `~/.claude/{CLAUDE.md,imports,scripts,skills}` may be symlinked to an external dotfiles repo (e.g. `~/dotfiles/claude-config/`) for version control — default `find` without `-L` would miss every real meta-work edit.

Detection MUST exclude noise paths: `logs/`, `cache/`, `.git/`, `backups/`, `projects/`, `sessions/`, `shell-snapshots/`, `tool-results/`, `scratch/precompact-snapshots/`, `plans/`, `__pycache__/`, plus files `.cheatsheet-last-rebuild`, `SKILLS_CHEATSHEET.md`, `SKILLS_CHEATSHEET.pdf`, `*.pyc`, `*.jsonl`, `*.log`.

#### Scenario: meta-work session in ~/Desktop still checkpoints

- **WHEN** CWD is `~/Desktop`
- **AND** session has touched `~/.claude/skills/spec/SKILL.md` within the last 30 minutes
- **AND** auto-compact is about to fire
- **THEN** pre-compact hook SHALL write a meta-snapshot to `~/.claude/scratch/precompact-snapshots/<ts>-meta-snapshot.md`
- **AND** snapshot SHALL list the recently touched files under `~/.claude/`

#### Scenario: symlink-following is mandatory

- **WHEN** `~/.claude/CLAUDE.md` is a symlink to `<external-dotfiles>/CLAUDE.md`
- **AND** user edits `<external-dotfiles>/CLAUDE.md` via the `~/.claude/` path
- **THEN** `find -L "$HOME/.claude" -type f -mmin -30` SHALL detect this edit
- **AND** meta-work mode SHALL activate

#### Scenario: noise excluded from meta-detection

- **WHEN** only `~/.claude/logs/foo.log` has been modified in the last 30 minutes
- **THEN** `is_meta_work` SHALL return false (not meta-work) because `*.log` and `logs/` are excluded

### Requirement: Skip-gate retreat

Both hooks SHALL silently exit when CWD indicates a non-OpenSpec workflow and no meta-work override applies. This prevents the defense layer from polluting unrelated projects.

#### Scenario: research_plan.json triggers skip

- **WHEN** CWD contains `research_plan.json`
- **AND** no meta-work edits detected in the last 30 minutes
- **THEN** both hooks SHALL exit 0 silently without injecting any message or writing any snapshot

#### Scenario: meta-analysis structure triggers skip

- **WHEN** CWD contains both `01_protocol/` and `09_qa/` directories
- **AND** no meta-work edits detected
- **THEN** both hooks SHALL exit 0 silently

#### Scenario: home root triggers skip unless meta-work

- **WHEN** CWD is exactly `$HOME` or `$HOME/Desktop`
- **AND** no meta-work edits detected
- **THEN** both hooks SHALL exit 0 silently

#### Scenario: non-openspec project with no retreat signal still exits

- **WHEN** CWD is a code project but has no `openspec/` directory
- **AND** no meta-work edits detected
- **THEN** pre-compact hook SHALL exit 0 silently (no snapshot destination)

### Requirement: Plan-mode bridge

A PostToolUse hook matching `ExitPlanMode` SHALL detect when the user has just approved a plan that implies spec-worthy scope and SHALL suggest `/opsx:propose` as a next step. The hook MUST NOT auto-invoke any tool or CLI — suggestion only.

#### Scenario: plan approval in openspec project suggests propose

- **WHEN** user approves a plan via `ExitPlanMode`
- **AND** CWD has `openspec/` present
- **THEN** hook SHALL inject a suggestion to consider `/opsx:propose <change-name>` before implementation begins

#### Scenario: plan approval without openspec stays silent

- **WHEN** user approves a plan via `ExitPlanMode`
- **AND** CWD has no `openspec/`
- **THEN** hook SHALL NOT inject any spec-related suggestion

### Requirement: Throttling and idempotence

The warn/strong hook SHALL use a per-session state file to guarantee each tier fires at most once per session. Re-entering the same tier (e.g. transcript size oscillating above/below the threshold) SHALL NOT re-fire the reminder.

#### Scenario: state file prevents re-fire

- **WHEN** warn tier has already fired in session `abc123`
- **AND** transcript size crosses `CONTEXT_WATCH_WARN_BYTES` again on a later prompt
- **THEN** hook SHALL detect the `warn` marker in `~/.claude/logs/context-watch-abc123.state` and SHALL NOT inject a second warn reminder

#### Scenario: new session resets throttle

- **WHEN** a new Claude Code session starts with a new `session_id`
- **THEN** no state file exists for that session and warn/strong tiers SHALL fire fresh on first threshold crossing
