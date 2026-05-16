<!-- BEGIN: spec skill (OpenSpec wrapper) — managed block, edit between markers OK -->
## OpenSpec Workflow（本專案）

@openspec/project.md

This project uses [OpenSpec](https://github.com/Fission-AI/OpenSpec) for spec-driven development. Lifecycle gates are wrapped by the global `spec` skill (`~/.claude/skills/spec/`). Above `@openspec/project.md` import pulls the project-level context（tech stack、IRB status、downstream hooks）automatically into every session — avoids re-loading via `/spec resume`.

### Retreat rules

If any of the following are detected, **stop OpenSpec workflow immediately** and route to the correct skill:

- `research_plan.json` present in this dir → use `research-plan` skill instead
- `01_protocol/` + `09_qa/` present → use `ma-end-to-end` skill instead

These exist because OpenSpec is wrong tool for statistical analyses and meta-analyses (those have their own structured workflows).

### Recommended pipeline

For non-trivial changes, prefer this order over ad-hoc edits:

```
/opsx:propose <change>      # write proposal.md / design.md / tasks.md
/opsx:apply                 # implement per tasks.md
/simplify                   # code-quality review (global skill)
/opsx:verify                # OpenSpec 3-dim check (completeness / correctness / coherence)
/verify                     # end-to-end check (global skill, e.g. Chrome MCP for web apps)
/opsx:archive               # merge delta into main specs (slash workflow has sync gate)
auto-git commit             # only after archive — see auto-git skill rules
```

**Skip steps only when**:
- Trivial typo / one-line fix → just edit, no propose
- User explicitly says "skip verify" / "just commit"

### Curator rules (hard)

- **Never** `git commit` without explicit user confirmation
- **Never** auto-write spec content — every requirement / scenario needs user-confirmed wording
- **Never** run `openspec archive --yes` raw CLI — always use `/opsx:archive` slash (it has a sync gate the raw CLI skips)
- PHI zero-tolerance: any patient identifier in spec → halt + suggest `phi-deid`

### Migration history

This skill replaced a self-built `spec.md` template (2026-04-19, 3-day trial). Decision rationale + rollback instructions: `~/.claude/skills/spec/MIGRATION.md`.
<!-- END: spec skill -->
