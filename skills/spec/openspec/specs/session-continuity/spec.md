# Spec: session-continuity

## Purpose

Turn OpenSpec's file system into cross-session working memory. Opening a new Claude Code session on an existing `openspec/` project SHALL reconstruct enough context (purpose, in-progress changes, recent decisions) from disk that the user does NOT have to re-explain history. Closing a session SHALL persist everything that matters back to disk through validated, user-confirmed artifacts.

The four user-invoked modes covered here — `resume`, `note`, `handoff`, `clear` — are the primary surfaces this capability exposes. They are deliberately thin wrappers around the OpenSpec CLI; all persistence lives in the `openspec/` tree on disk.

## Requirements

### Requirement: Resume mode context warm-up

`/spec resume` SHALL load a fixed bundle of files into the model's context on session start so that subsequent turns do NOT require re-asking the user about prior decisions or prior state.

The bundle SHALL include: `openspec/project.md` (full), every in-progress change's `proposal.md` + `design.md` (full), every capability spec touched by those in-progress changes (full), the first-paragraph summary of the 3 most recent archived changes, and the 3 most recent files under `openspec/decisions/` if that directory exists.

The warm-up cost of 1–3K tokens is intentional. The capability MUST NOT attempt to "optimize" by reading partial files or summaries only — full-file reads are what allow the model to answer "why did we decide X" without asking the user.

#### Scenario: resume reads full warm-up bundle

- **WHEN** user runs `/spec resume` in a CWD with `openspec/` present
- **AND** `openspec list --json` reports 2 in-progress changes
- **THEN** system SHALL Read `openspec/project.md`, both `proposal.md` + `design.md` files for each in-progress change, every capability spec referenced by those changes, and the 3 most recent `archive/` summaries
- **AND** system SHALL output a 3-sentence summary: Project / Last known state / Suggested next

#### Scenario: resume flags empty project.md

- **WHEN** `openspec/project.md` exists but contains only template placeholders
- **THEN** resume summary SHALL flag "project.md not yet filled" and suggest completing Purpose / Stack before proceeding

#### Scenario: resume with no in-progress changes

- **WHEN** `openspec list` returns zero non-archived changes
- **THEN** resume summary SHALL still read `project.md` + recent archives + recent decisions
- **AND** "Suggested next" sentence SHALL propose `/opsx:propose` to start a new change

### Requirement: Note mode decision routing

`/spec note "<text>"` SHALL capture ad-hoc in-conversation decisions to disk with a low-cost code path (no proposal/design/tasks overhead). The routing destination SHALL be deterministic based on current in-progress change count.

#### Scenario: zero in-progress changes routes to decisions dir

- **WHEN** `openspec list` reports zero non-archived changes
- **AND** user runs `/spec note "use Postgres not SQLite, reason: FTS"`
- **THEN** system SHALL append to `openspec/decisions/<YYYY-MM-DD>.md` (creating the file with `# Decisions — <date>` header if absent)
- **AND** entry SHALL be formatted as `## HH:MM — <auto-tag>\n\n<note body>`

#### Scenario: exactly one in-progress change routes to its design.md

- **WHEN** `openspec list` reports exactly one non-archived change
- **AND** user runs `/spec note "<text>"`
- **THEN** system SHALL append to `openspec/changes/<change>/design.md` under a `## Decisions` section
- **AND** entry SHALL be formatted as `### YYYY-MM-DD HH:MM — <auto-tag>\n\n<note body>`

#### Scenario: multiple in-progress changes asks user

- **WHEN** `openspec list` reports ≥ 2 non-archived changes
- **AND** user runs `/spec note "<text>"`
- **THEN** system SHALL use AskUserQuestion listing all in-progress change names plus option "none, write to openspec/decisions/"
- **AND** SHALL route based on user's selection

#### Scenario: empty note text is rejected

- **WHEN** user runs `/spec note` with no argument AND then provides empty string via follow-up prompt
- **THEN** system SHALL abort with message "note text required" and SHALL NOT create any file

### Requirement: Handoff mode validate and archive

`/spec handoff` SHALL run `openspec validate --all`, detect completed-but-unarchived changes, offer each for archival via AskUserQuestion, and SHALL prompt for a git commit as a hard interaction point that cannot be silently skipped.

#### Scenario: validate errors block handoff

- **WHEN** `openspec validate --all` exits non-zero
- **THEN** system SHALL report the validation errors and SHALL NOT proceed to archive prompting until errors are resolved

#### Scenario: completed change offered for archive

- **WHEN** an in-progress change has all 4/4 artifacts and all `tasks.md` checkboxes ticked
- **THEN** system SHALL ask via AskUserQuestion "archive <change-name>?"
- **AND** on confirmation SHALL recommend `/opsx:archive <change>` (not raw `openspec archive --yes`, which skips the sync gate)

#### Scenario: dirty git state forces commit decision

- **WHEN** `git status` reports any uncommitted changes after archive step
- **THEN** system SHALL use AskUserQuestion with exactly 3 explicit options: (1) commit now via auto-git, (2) skip commit and handle manually, (3) cancel handoff
- **AND** MUST NOT silently skip this prompt
- **AND** MUST NOT invoke `git commit` directly or with `--no-verify` if user selects option 1 — route through the auto-git skill

### Requirement: Clear mode never auto-invokes /clear

`/spec clear` SHALL run the full handoff flow and SHALL then print an explicit instruction asking the user to type `/clear` manually. The system SHALL NOT attempt to invoke `/clear` itself, because Claude Code treats slash commands as user-only intent and skills cannot invoke them on behalf of the user.

#### Scenario: clear prints manual /clear instruction

- **WHEN** `/spec clear` completes handoff steps (validate, archive offers, commit prompt)
- **THEN** final output SHALL include the exact sentence "現在請手動執行：/clear" (or English equivalent)
- **AND** system SHALL NOT emit a tool call attempting to invoke `/clear`

#### Scenario: clear scans for pending in-conversation decisions

- **WHEN** `/spec clear` is invoked
- **AND** the current conversation contains user-confirmed decisions that were NOT captured via `/spec note` or archived change docs
- **THEN** system SHALL list up to 5 such decisions and ask "should we `/spec note` these before clearing?" before proceeding to handoff steps

#### Scenario: clear with clean state skips middle steps

- **WHEN** `/spec clear` is invoked
- **AND** there are no in-progress changes, no dirty git state, and no uncaptured decisions
- **THEN** system SHALL skip the handoff middle steps and proceed directly to the manual `/clear` instruction

### Requirement: Curator mode — no silent writes

All four modes in this capability SHALL follow curator discipline: non-trivial writes require user confirmation via AskUserQuestion. The system SHALL NOT auto-commit, auto-archive, auto-overwrite, or auto-delete.

#### Scenario: handoff never auto-commits

- **WHEN** `/spec handoff` reaches the commit decision point
- **THEN** system SHALL use AskUserQuestion (never proceed without explicit user selection)

#### Scenario: note never overwrites existing entries

- **WHEN** `/spec note` appends to `openspec/decisions/<date>.md` or `design.md`
- **THEN** system SHALL append (not overwrite) and SHALL preserve all existing content in the target file
