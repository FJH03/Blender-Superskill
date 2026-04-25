# Blender SuperSkill

`Blender SuperSkill` is a Codex skill for building better Blender assets through disciplined, reference-first modeling instead of low-quality image-to-3D mesh dumps.

It is aimed at indie and cash-strapped developers who want a stronger AI-assisted Blender workflow for:

- hard-surface props
- buildings and utility assets
- lattice towers, pylons, and scaffold-like structures
- cleanup, critique, and iterative repair passes
- deterministic scripted modeling runs through the included orchestrator

## What It Includes

- `skills/blender-superskill/SKILL.md` - the core skill instructions
- `skills/blender-superskill/references/` - reusable modeling patterns and failure notes
- `skills/blender-superskill/scripts/blender_orchestrator.py` - a stepwise orchestration helper for complex reference-driven runs
- `skills/blender-superskill/agents/openai.yaml` - Codex UI metadata

## Install

Install the skill from the repo path:

- repo: `powerhouse90/Blender-Superskill`
- path: `skills/blender-superskill`

You can either:

1. Ask Codex to install the skill from that GitHub repo/path.
2. Copy `skills/blender-superskill` into your local `~/.codex/skills/` directory manually.

Restart Codex after installing so the skill is picked up cleanly.

## What This Skill Is For

This skill is built around one core idea: AI is much more useful when it helps construct assets intentionally inside Blender than when it hands you a messy mesh that looks right only from one angle.

The skill pushes for:

- anatomy and proportion planning before modeling
- anchor-based placement instead of eyeballing
- close-up interface checks where parts are supposed to touch
- modular repeated parts with measured duplication
- inspection and repair loops instead of one-shot generation

## Requirements

- Codex with Blender MCP access
- Blender workflows where the model can inspect viewports, objects, and scene structure

The orchestrator script is optional, but especially useful for larger reference-driven assets where step discipline matters.

## License

MIT
