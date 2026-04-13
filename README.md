# GitHub Operator Set

GitHub operations skill pack for OpenClaw, centered on repo bootstrap and pre-push publication gates.

## What this repository contains

This repository is a focused export of the GitHub team layer from a larger OpenClaw workspace.
It is intended to be a small, publishable operator set rather than a full workspace mirror.

Included components:
- `github-ops` — top-level intake and routing entrypoint for GitHub work
- `github-repo-creator` — local project to GitHub repository bootstrap workflow
- `pre-push-gate` — pre-push / pre-PR publication safety gate

## Why this repo exists

This repo exists to validate the GitHub team structure as an independent publication unit.
The immediate goal is to verify that repo bootstrap and publication safety workflows are coherent in isolation.

## Repository structure

```text
skills/
├── github-ops.SKILL.md
├── github-repo-creator.SKILL.md
└── pre-push-gate.SKILL.md
```

## Scope

Included:
- GitHub team routing
- repo bootstrap workflow
- publication safety gate design

Excluded:
- the rest of the OpenClaw workspace
- unrelated local operating documents
- broader coding-team and investing-team assets

## Publication status

Current state:
- private validation first
- public release only after pre-push checks and repo-level audit pass

## Notes on export format

The files are exported as `*.SKILL.md` artifacts to make the skill boundaries explicit in a standalone repository.
This repository prioritizes clarity of role boundaries over full runtime packaging.

## License

MIT
