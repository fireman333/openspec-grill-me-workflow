# Spec: assessment-and-bootstrap

## Purpose

Decide whether a given project SHOULD adopt OpenSpec, and if yes, bootstrap OpenSpec structure cleanly. This capability is the gate that prevents OpenSpec from mis-firing on non-coding projects (RA / 國考 / meta-work) while surfacing it for projects where it actually adds value.

## Requirements

### Requirement: Rubric-based assessment

The system SHALL evaluate every project against a weighted rubric before recommending `/spec init` or `/spec retro`. Raw heuristics (e.g. "has package.json") MUST NOT auto-trigger OpenSpec adoption without score aggregation.

#### Scenario: coding project crosses threshold

- **WHEN** CWD contains `package.json` with React/Vue/Svelte/Next/Vite/Astro dependencies **AND** `.git/` with ≥5 commits **AND** ≥10 source files
- **THEN** rubric score ≥ 5 and system SHALL recommend `/spec init`

#### Scenario: one-off script stays below threshold

- **WHEN** CWD contains a single Python script and no tests
- **THEN** rubric score ≤ 1 and system SHALL recommend skipping OpenSpec

#### Scenario: edge-case middle score

- **WHEN** rubric score is 2–4
- **THEN** system SHALL list positive and negative signals and let the user decide (not auto-recommend either way)

### Requirement: Retreat rules

The system SHALL unconditionally refuse to recommend OpenSpec in projects that belong to other structured workflows. Retreat signals have −10 weight to dominate any positive signals.

#### Scenario: statistics project retreat

- **WHEN** CWD contains `research_plan.json`
- **THEN** system SHALL NOT recommend OpenSpec and SHALL point the user toward `research-plan` skill

#### Scenario: meta-analysis project retreat

- **WHEN** CWD contains both `01_protocol/` and `09_qa/` directories
- **THEN** system SHALL NOT recommend OpenSpec and SHALL point the user toward `ma-end-to-end` skill

#### Scenario: meta-work on ~/.claude/ itself

- **WHEN** CWD is `~/.claude/` or its subdirectory
- **AND** the project is not a deliberate skill cluster (≥3 interdependent skills)
- **THEN** system SHALL apply −3 meta-work penalty (not recommend OpenSpec at the directory level)
- **AND** the meta-work exception in `compact-defense` still applies to protect in-session decisions

### Requirement: Skill-cluster boost

The system SHALL recognize skill clusters as high-value OpenSpec candidates. A cluster is defined as ≥3 interdependent skills / modules sharing hook scripts, a shared contract, or an ecosystem documentation file.

#### Scenario: OE cluster qualifies

- **WHEN** a project organizes ≥3 related skills (e.g. `openevidence`, `audit-oe`, `oe-triangulate`) with a shared `ECOSYSTEM.md`
- **THEN** rubric SHALL add +4 to the score
- **AND** `openspec/` SHALL be placed at the cluster root (not inside individual skills)

#### Scenario: cluster init overrides meta-work penalty

- **WHEN** the skill-cluster signal fires (+4) in a CWD under `~/.claude/skills/`
- **THEN** the CWD-under-~/.claude/ penalty (−3) is waived

### Requirement: CLI preflight

Any mode that invokes the `openspec` CLI (init / retro / resume / handoff) MUST verify CLI availability before proceeding. Silent degradation is forbidden.

#### Scenario: CLI missing

- **WHEN** the user invokes `/spec init` and `which openspec` returns nothing
- **THEN** system SHALL tell the user "OpenSpec CLI not installed" with the install command `npm install -g @fission-ai/openspec@latest` (Node ≥ 20.19)
- **AND** ask via AskUserQuestion whether to install now
- **AND** MUST NOT fall back to manual file creation

### Requirement: Bootstrap artifacts

`/spec init` SHALL produce three authoritative files beyond what `openspec init` creates natively. These are the wrapper's additions for curator workflow and project-level context propagation.

#### Scenario: init creates project.md

- **WHEN** `/spec init` completes
- **THEN** `openspec/project.md` exists with Purpose / Stack / Out-of-scope sections filled from user answers

#### Scenario: init creates config.yaml with curator rules

- **WHEN** `/spec init` completes
- **THEN** `openspec/config.yaml` exists with curator rules (no auto-commit, PHI zero-tolerance, retreat rules for `research_plan.json` / `01_protocol/`)

#### Scenario: init appends to project CLAUDE.md

- **WHEN** `/spec init` completes and project CLAUDE.md does not contain `BEGIN: spec skill` marker
- **THEN** system SHALL append the managed block from `templates/project-claude-md-append.md` to `<project>/CLAUDE.md`
- **AND** the managed block SHALL include `@openspec/project.md` auto-import directive so every session pre-loads project context

#### Scenario: init preserves existing managed block

- **WHEN** `<project>/CLAUDE.md` already contains `BEGIN: spec skill` marker
- **THEN** system SHALL ask via AskUserQuestion whether to overwrite the managed block (never silent overwrite)

### Requirement: Git-history retro

`/spec retro` SHALL harvest spec requirements from git tag + release notes + commit history for mature projects, as an alternative to propose-from-zero. Retro writes only to `openspec/specs/<capability>/`, never to `openspec/changes/`.

#### Scenario: mature project qualifies for retro

- **WHEN** CWD has `.git/` with ≥3 git tags **AND** no existing `openspec/`
- **THEN** `/spec retro` is the preferred entry point over `/spec init` → `/opsx:propose`

#### Scenario: retro output is user-reviewable

- **WHEN** `/spec retro` extracts requirements from commit history
- **THEN** system SHALL present the draft (candidate capabilities + per-capability requirements + BDD scenarios) for user review via AskUserQuestion BEFORE writing to disk

#### Scenario: retro validates after write

- **WHEN** `/spec retro` finishes writing `openspec/specs/<capability>/spec.md` files
- **THEN** system SHALL run `openspec validate --all` and fix any errors before reporting success
