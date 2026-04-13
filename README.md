# GitHub Operator Set

GitHub operations skill pack for OpenClaw, centered on repo bootstrap and pre-push publication gates.

## Included Skills

- `github-ops` — top-level intake and routing entrypoint for GitHub work
- `github-repo-creator` — local project to GitHub repository bootstrap workflow
- `pre-push-gate` — pre-push / pre-PR publication safety gate

## Why this repo exists

This mini-repo is an exportable GitHub team slice from a larger OpenClaw workspace.
It exists to validate the GitHub team structure as an independent publication unit before broader release.

## Structure

```text
skills/
├── github-ops.SKILL.md
├── github-repo-creator.SKILL.md
└── pre-push-gate.SKILL.md
```

## Notes

- This repository is intentionally minimal.
- It focuses on GitHub operator workflows rather than the whole OpenClaw workspace.
- Public release should happen only after private validation of the repo creation and publication flow.

## License

MIT
