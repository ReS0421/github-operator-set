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


## Usage guide

This repository is best used as a design and export reference for assembling a GitHub-focused operator layer inside OpenClaw or a similar agent workspace.

Typical use cases:
- add `github-ops` as the top-level GitHub intake skill
- use `github-repo-creator` when exporting a local project into a GitHub repository
- use `pre-push-gate` before push, PR creation, or publication events

## Adoption guide

### 1. Start with routing
Treat `github-ops` as the default entrypoint when a user asks for GitHub-related work but has not named the exact specialist.

### 2. Keep specialists separate
Do not collapse all GitHub behaviors into one mega-skill.
Recommended separation:
- routing: `github-ops`
- repo bootstrap: `github-repo-creator`
- publication gate: `pre-push-gate`
- direct GitHub operations: `github`
- issue automation: `gh-issues`

### 3. Use staged rollout
Recommended rollout order:
1. private repo validation
2. pre-push / publication gate validation
3. public release
4. branch protection and contribution policy hardening

### 4. Treat publication as a gate, not a side effect
`pre-push-gate` should remain an explicit decision point before remote publication.
That keeps repo creation flows and publication safety checks understandable.

## Suggested integration pattern

```text
user request
  ↓
github-ops
  ├── github-repo-creator   (repo bootstrap)
  ├── pre-push-gate         (publication safety)
  ├── github                (direct operations)
  └── gh-issues             (issue automation)
```

## Contribution notes

If you extend this repository:
- preserve role boundaries between router and specialists
- prefer explicit gate phases over implicit publication
- keep documentation examples safe for secret scanners
- validate changes in private mode before broadening exposure

## License

MIT
