# `skills/` — `/spec` and `/grill-me` source

This directory contains the actual [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) source for `/spec` (OpenSpec lifecycle wrapper) and `/grill-me` (structured clarification grilling) — the two skills described in the [workflow notes](../README.md) at the repo root.

> 🛠 **License**: skills here are released under **MIT** (see [`LICENSE.MIT`](LICENSE.MIT)). The root-level workflow notes / diagrams remain under CC-BY-4.0.

---

## ⚠️ Read this before installing — "personal setup" disclaimer

These skills evolved organically inside one person's Claude Code setup. They **work**, but they were never designed for general distribution. Expect to adapt before you get value:

### What you'll see that you don't have

The `/spec` skill router and `/grill-me` retreat rules reference several **other personal skills that are NOT published in this repo**:

| Reference in source | What it is | If you don't have it |
|---|---|---|
| `/research-plan`, `/rp` | Personal skill for medical clinical research statistical pipelines (Table 1, Kaplan-Meier, Cox, etc.) | Comment out the retreat / cross-reference lines, or replace with your own equivalent |
| `/ma-end-to-end` | Personal end-to-end meta-analysis pipeline skill | Same — comment / replace |
| `/cps`, `/cpi` | Clinical problem solving / clinical problem investigation skills | Same |
| `/medq`, `/oe`, `/lit-review` | Personal literature search + OpenEvidence wrapper skills | Same |
| `/verify`, `/go` | Personal task verification skill | Same |
| `/auto-git`, `/auto-skill-eval`, `/no-wheels` | Personal automation skills | Same |
| Hook scripts `~/.claude/scripts/spec-*.sh` (session-start / context-watch / pre-compact) | Referenced by `/spec`'s "Three-layer defense" section, not bundled here | Either write your own based on the spec'd behavior, or skip the hooks (the skill still works without them, you just lose proactive context-watch warnings) |

**You have two paths**:

1. **Study, don't install** — read the SKILL.md files as a reference design, write your own version tailored to your stack.
2. **Install, then adapt** — copy into `~/.claude/skills/`, then comment out the cross-references that don't apply and re-add your own retreat rules.

I do **not** recommend installing as-is without reading first.

### Hook scripts are NOT included

The `/spec` skill describes three hook scripts:
- `~/.claude/scripts/spec-session-start.sh`
- `~/.claude/scripts/spec-context-watch.sh`
- `~/.claude/scripts/spec-pre-compact.sh`

These provide the three-layer auto-compact defense (predictive warning → checkpoint snapshot → handoff prompt). They're behavior is fully spec'd in [`spec/openspec/specs/compact-defense/spec.md`](spec/openspec/specs/compact-defense/spec.md) — you can implement your own from that spec.

I haven't published my actual hook scripts here because they're tightly coupled to my dotfiles layout and would mislead more than help.

---

## Install (if you want to anyway)

```bash
# 1. Copy the skills into your Claude Code skills directory
cp -r skills/spec       ~/.claude/skills/
cp -r skills/grill-me   ~/.claude/skills/

# 2. (Optional) install OpenSpec CLI — required for /spec's init / propose / archive
npm install -g @fission-ai/openspec@latest

# 3. Restart Claude Code so the skills register
```

Then `/spec` and `/grill-me` (or aliases `/spec`, `/grill`) become available.

You'll want to also:
- Add a Skill Router stub in your `~/.claude/CLAUDE.md` so the slash commands trigger the skills (the SKILL.md headers explain the trigger keywords)
- Read `spec/SKILL.md` § "與其他 skill 的互動" and comment out references to skills you don't have
- Read `grill-me/SKILL.md` § "退讓優先" and adjust the medical-domain retreat rules to your domain

---

## File layout

```
skills/
├── README.md                     # this file
├── LICENSE.MIT                   # MIT license for skills/ subfolder
├── spec/
│   ├── SKILL.md                  # main skill spec (612 lines, includes assess/init/clarify/retro/resume/handoff/note/clear modes)
│   ├── templates/
│   │   ├── project.md            # template for openspec/project.md
│   │   ├── config.yaml.tmpl      # template for openspec/config.yaml
│   │   └── project-claude-md-append.md  # injected into <project>/CLAUDE.md by /spec init
│   ├── references/
│   │   └── planning-patterns.md  # Manus-style patterns referenced by /spec resume
│   └── openspec/                 # /spec dogfoods itself! real openspec/ folder for the spec skill
│       ├── project.md
│       └── specs/
│           ├── assessment-and-bootstrap/spec.md
│           ├── session-continuity/spec.md
│           └── compact-defense/spec.md
└── grill-me/
    ├── SKILL.md                  # main skill (354 lines, Quick + Deep modes)
    └── packs/                    # 5 facet packs for different task types
        ├── software-project.md   # SE projects (used by /spec clarify in Deep mode)
        ├── content-creation.md   # blog posts, essays, threads
        ├── learning-plan.md      # study / curriculum design
        ├── life-decision.md      # major life choices
        └── generic.md            # fallback for anything else
```

---

## Why two skills?

- **`/grill-me`** is a general clarification tool: given any half-formed plan, it asks 5–7 (Quick) or 15+ (Deep) structured questions to expose unstated assumptions. Domain-agnostic.
- **`/spec`** is the OpenSpec lifecycle wrapper: it handles assess → init → resume → handoff lifecycle around the [OpenSpec CLI](https://github.com/Fission-AI/OpenSpec).

They compose: `/spec clarify` is a thin wrapper that invokes `/grill-me` in Deep mode with the `software-project` pack and writes output to `openspec/project-clarifications.md`, which `/spec init` then auto-reads to pre-fill `project.md`.

See the [root README](../README.md) for the full integration design.

---

## License

Skills in this directory: **MIT** ([LICENSE.MIT](LICENSE.MIT)).

Workflow notes / diagrams / hero image at the repo root: **CC-BY-4.0** ([../LICENSE](../LICENSE)).

Third-party dependencies referenced by the skills (OpenSpec, Claude Code) retain their own licenses.
