---
name: git-workflow
description: "Invoke before any git operations — branching, committing, merging, or creating pull requests. Contains project-specific branch naming, commit format, and merge strategy."
---

# Git Workflow

## Detected
- Default branch: main
- Contributors: 69
- Ana CLI: pipeline artifacts committed via `ana artifact save` with [slug] prefix. Build agent creates `{branchPrefix}{slug}` branches (read `branchPrefix` from `.ana/ana.json`, default `feature/`). Co-author from ana.json.

## Rules
- Commit each logical change separately. Don't batch unrelated changes into one commit.
- Commit messages use lowercase scope prefix style, not conventional commits. Examples: `email: handle provider error spikes`, `Fix Gmail draft threading`, `Add inbox label removal action`. Include PR number in parens when merging.
- Stage specific files for each commit. Avoid `git add .` or `git add -A` — review what you're committing.
- Pre-commit runs lint-staged (Biome formatting only). Pre-push runs gitleaks secret scanning. No typecheck or test gates in hooks — those run in CI.
- Merge strategy is merge commits (not squash). Branch names use fix/, feat/, or username/ prefixes.

## Gotchas
*Not yet captured. Add as you discover them during development.*

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
