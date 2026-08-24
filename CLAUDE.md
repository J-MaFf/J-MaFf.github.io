# Project Instructions for AI Agents

This file provides instructions and context for AI coding agents working on this project.

<!-- BEGIN BEADS INTEGRATION (reconciled with git-policies skill, see below) -->
## Beads Issue Tracker

This project uses **bd (beads)** as the execution/memory layer *underneath* GitHub Issues —
see the `git-policies` skill's "Beads (`bd`) — Layered Task & Memory Workflow" section for the
full convention this repo follows. Run `bd prime` to see workflow context and commands.

**Layered model:** the GitHub Issue is the shippable unit (branch → PR → `Fixes #N` →
squash-merge, human-gated). Beads is the fine-grained execution/memory layer underneath it —
the `bd ready` queue, the dependency graph, and `bd remember` persistent memory.

### Quick Reference

```bash
bd ready               # Find available work
bd show <id>           # View issue details
bd update <id> --claim  # Claim work
bd close <id>          # Complete work
bd dolt push            # Sync bead state immediately after any create/update/close
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists.
- Run `bd prime` for detailed command reference.
- Use `bd remember` for **repo-scoped** persistent knowledge (things that should travel with
  this repo, not the user). It does not replace the global Claude memory system, which stays
  the home for cross-repo / user-level context.
- `bd dolt push` runs automatically after every bead mutation — no confirmation needed. It
  syncs `refs/dolt/data`, which is not covered by the Main Branch Ruleset on `refs/heads/main`,
  so it carries none of the review/control weight of a `git push` to a protected branch.

**Architecture in one line:** issues live in a local Dolt DB; sync uses `refs/dolt/data` on your git remote; `.beads/issues.jsonl` is a passive export, not the sync channel. See https://github.com/gastownhall/beads/blob/main/docs/SYNC_CONCEPTS.md for details and anti-patterns.

## Session Completion

At session close, make work durable **without** merging — beads syncs and code commits stay
independent of the human-gated merge:

1. **Sync beads** — `bd dolt push` (should already be current if pushed per-mutation above).
2. **Commit and push the feature branch** — never commit or push directly to `main`:
   ```bash
   git add <files> && git commit -S -m "..."   # signed commit, on the FEATURE branch
   git push -u origin <feature-branch>
   ```
3. **Open or update the PR** referencing the GitHub issue (`Fixes #N`) and **stop at the merge
   gate** — a human approves the squash-merge to `main`. Never auto-merge.

**Critical rules:**
- Explicit user, repository, or orchestrator instructions override this Beads block.
- `bd dolt push` is not merge-adjacent and does not require confirmation; a `git push` to
  `main` (or a merge) always does.
- If a required sync or push is blocked, stop and report the exact command and error.

> This block was manually reconciled with the `git-policies` skill's Beads convention on
> 2026-08-24 (see J-MaFf/J-MaFf.github.io#8). **Do not run `bd setup claude` again** — it
> reverts this reconciliation back to bd's default template. `bd setup claude --check` will
> keep reporting this block as "stale"; that is expected.
<!-- END BEADS INTEGRATION -->


## Build & Test

_Add your build and test commands here_

```bash
# Example:
# npm install
# npm test
```

## Architecture Overview

_Add a brief overview of your project architecture_

## Conventions & Patterns

_Add your project-specific conventions here_
