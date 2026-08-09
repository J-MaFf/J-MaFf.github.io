<!-- GENERATED FILE — do not edit by hand.
     Source of truth: J-MaFf/claude-skills · git-policies/SKILL.md
     Published by: claude-skills/.github/workflows/publish-git-policies.yml
     Edit the skill, not this file. -->

# Git Policies & GitHub Conventions

Personal global standards for git workflow, branch management, PR conventions, and GitHub issue tracking.

---

## Development Workflow

**Always create a GitHub issue BEFORE implementing a bug fix or feature.** Never implement first and document later.

The workflow:

1. **Spot a bug or issue** → Create a GitHub issue first
   - Use appropriate labels: `bug`, `enhancement`, `documentation`, or other GitHub defaults
   - Always include `--assignee J-MaFf`
   - Example: `gh issue create --title "Fix auth timeout" --label bug --assignee J-MaFf`
   - Before picking work, list what's already open: `git issues` (current repo) or `git issues --all` (across all your repos)

2. **Create a branch** tied to that issue with a logically descriptive name (see **Branch Naming** below)

3. **Implement on that branch**
   - Write meaningful commit messages with all relevant details (see **Commit Messages** below)
   - Each commit should be self-documenting, even though it will be squashed later — the details inform the final squash message

4. **Open a PR** referencing the issue
   - Use `Fixes #N` in the PR body (where N is the issue number)
   - Include `--assignee J-MaFf` and `--label <appropriate-label>` in the `gh pr create` command
   - Example structure: `gh pr create --assignee J-MaFf --label enhancement --body "Fixes #42\n\nImplements feature..."`

5. **Keep branch in sync** with main
   - Rebase your branch onto main regularly: `git rebase main`
   - This keeps history clean and avoids merge commits
   - If conflicts arise, resolve and continue rebase

6. **Wait for merge approval** — never auto-merge unless explicitly told "merge please" or the user is busy/away

---

## Branch Naming Conventions

Use kebab-case with a type prefix:

- **Bug fixes:** `fix/<short-description>` 
  - Example: `fix/auth-timeout-issue`
  
- **Features:** `feat/<short-description>`
  - Example: `feat/dark-mode-toggle`
  
- **Documentation:** `docs/<short-description>`
  - Example: `docs/api-endpoint-guide`

---

## Commit Messages During Development

Each commit on your branch should include all relevant details, even though commits will be squashed when merging. These details become the working material for your final squashed commit message.

**Format:**

```
<type>: <short summary>

<detailed explanation of what changed and why>

- Related files or functions modified
- Dependencies or side effects
- Testing approach
```

**Example:**

```
feat: Add dark mode toggle to header

Extracted color theme logic into a separate ThemeProvider component
to allow theme switching at the app level. Integrated with existing
CSS variables system and added localStorage persistence.

- Modified: Header.tsx, ThemeProvider.tsx, styles/colors.css
- Dependencies: Added react-context for theme state
- Tested: Manual toggle in Chrome/Safari, localStorage persistence verified
```

**Why this matters:** These detailed commits let you review what was actually done and extract the meaningful steps for your final squashed commit message.

---

## Rebase Workflow

Keep your feature branch in sync with main by rebasing, not merging:

```bash
git rebase main
```

**When to rebase:**
- Before opening a PR (ensure it applies cleanly to main)
- If main moves on while your PR is under review (rebase and force-push)
- To keep history linear and clean

**Conflict resolution:**
- Resolve conflicts in the affected files
- Continue rebase: `git rebase --continue`
- If something goes wrong, abort: `git rebase --abort`

**Force push after rebase:**
- After rebasing a feature branch, you'll need to force-push: `git push --force-with-lease origin <branch>`
- Use `--force-with-lease` (safer than `--force`) — it prevents overwriting others' work if branch was pushed
- **Only force-push to feature branches, never to `main` or protected branches**

---

## Parallel Branches & Integration Waves

When several branches are open at once (multiple devs, or a fleet of agents), pick the
integration pattern **before** opening the PRs:

**Independent changes → parallel PRs + sequential bottom-up merge.**
Merge one PR at a time; after each merge, rebase the next branch onto fresh `main`
(`git rebase main`, then `git push --force-with-lease`), let required checks re-run on the
rebased head, merge, repeat. Once the human has approved merging the wave, use GitHub
auto-merge so each merge fires the moment its checks pass instead of polling:

```bash
gh pr merge <n> --squash --auto --subject "..." --body "..."
```

(`--auto` requires the repo setting `allow_auto_merge=true` — see New Repository Setup.
This does not conflict with the "never auto-merge" rule: the human approves the merge;
`--auto` only lets CI gate the *timing*.) Never merge a rebased branch whose required
checks haven't re-passed on the new head SHA.

**Prefer a merge queue when available.** GitHub's merge queue does the above
automatically: PRs are enqueued, speculatively rebased onto the latest candidate state,
CI runs on the *result*, and green PRs land in order — nobody hand-rebases the same
branch twice. Merge queues currently require organization-owned repos; on personal
(user-owned) repos, the auto-merge + sequential-rebase loop above is the substitute.
Even for a solo dev, agent fleets create real parallel-PR contention, so enable the
queue wherever the repo qualifies.

**Genuinely dependent changes → stack the PRs.**
Base each branch on the previous branch, not `main`, and target the PR at that base
(`gh pr create --base <prev-branch>`). Merge the stack **bottom-up** with squash; with
`delete_branch_on_merge` enabled, GitHub retargets each still-open PR to `main` when its
base branch is deleted, and `Fixes #N` closes the issue when the retargeted PR lands on
the default branch. Only stack when the work is truly sequential — an unnecessary stack
blocks PR N on PR 1's review. A stacked chain also avoids all cross-PR conflicts by
construction, which makes it the right shape for agent fleets working on overlapping files.

**Squash-merged stacks require a restack after every merge.** GitHub's retarget changes
the PR's base ref but does NOT rebase the branch: after the parent squash-merges, the
child's lineage still carries the parent's *original* commits while `main` carries an
equivalent *squash* commit, and git conflicts wherever parent and child touched the same
hunk regions (a superset hunk still conflicts — identical content does not "net out" in
a merge). After each parent merges, **restack the child by rebasing only its own commits
onto main** — `git rebase --onto main <old-parent-tip> <child-branch>` — then force-push
with lease, let required checks re-pass on the new head, and merge. Never merge a
stacked PR whose base just changed without confirming it is MERGEABLE against `main`;
and never let merge automation fall through to `gh pr merge` when the base/mergeable
check did not explicitly pass (a merge issued while the PR still targets its old base
lands the squash on the *parent branch*, not `main`).

**Merge style does not affect conflicts.** Conflicts come from content overlap, not from
squash vs merge-commit vs rebase-merge. Keep the squash default; the pattern choice above
is what controls integration pain.

---

## Minimizing Cross-Branch Conflicts

Most cross-branch conflicts come from a few shared "magnet" files. Design them away
rather than resolving the same conflict N times:

- **CHANGELOG.md is a conflict magnet.** Every PR appends to the same `[Unreleased]`
  block, so any two open PRs collide there. The standing resolution is **keep-both**
  (union of all bullets from both sides), never keep-one. If a repo sees frequent
  parallel waves, *consider* changelog fragments (each PR adds a uniquely-named file
  under `changelog.d/`, assembled into CHANGELOG.md at release) — but that is an
  opt-in per-repo decision, not the default; the Keep-a-Changelog single-file format
  remains the standard until a repo explicitly adopts fragments.
- **Committed generated files guarantee conflicts.** A generated artifact checked into
  the repo collides on every pair of PRs that touch its source. Prefer building it in CI
  and publishing it as a release asset. When it must stay committed (e.g. a raw-URL
  one-liner depends on it), the conflict resolution is mechanical and must be treated
  that way: **resolve the source files, regenerate the artifact with the build script,
  stage the regenerated output** — never hand-merge generated content. Marking it
  `linguist-generated=true` in `.gitattributes` collapses it in PR review but does not
  prevent conflicts.
- **Short-lived branches beat clever merging.** Integration pain scales with how long
  branches diverge. Keep PRs small and land them within a day or two; hide unfinished
  work behind flags rather than long-lived branches.

If a repo has an unavoidable magnet file, document it (and the scripted resolution) in
that repo's CLAUDE.md or STATUS.md so agents resolve it consistently.

---

## Git Aliases

Custom aliases are defined in `C:/Users/7maff/Documents/Scripts/gitconfig/.gitconfig.template`
(the gitconfig repo is the source of truth). Run `git alias` to print the full, current list —
prefer that over this table if they ever disagree. Workflow-relevant aliases:

| Alias | Expands to / does |
|---|---|
| `git s` | `status -sb` |
| `git lg` | `log --graph --oneline --decorate --all` |
| `git last` | Last commit with stats |
| `git recent` | 15 most recently committed local branches |
| `git nb <name>` | `switch -c <name>` (new branch) |
| `git sync` | `pull --rebase --autostash` |
| `git pushf` | `push --force-with-lease` (never plain `--force`) |
| `git amend` / `git reword` | Amend last commit without / with editing message |
| `git undo` | `reset --soft HEAD~1` |
| `git unstage` | `reset HEAD --` |
| `git start` | Helper: begin work (issue/branch setup) |
| `git main` | Helper: switch to main and update it |
| `git cleanup` | Helper: delete branches whose remotes are gone (`--force` also deletes local-only) |
| `git branches` | Fetch and create local tracking branches for all remote branches |
| `git pr` / `git prs` | `gh pr view --web` / `gh pr status` |

Use these when they fit the step (e.g. `git sync` before branching, `git pushf` after a rebase,
`git cleanup` after a merge) — they encode the conventions in this document.

---

## Branch Cleanup

After a PR merges, clean up local and remote branches using your git aliases:

```bash
git cleanup          # Delete branches with deleted remotes (merged)
git cleanup --force  # Also delete local-only branches
```

Manage remote tracking branches with:

```bash
git branches         # Download all remote branches and create local tracking branches
```

---

**Always sign commits** using `git commit -S`. Signing is required — never fall back to unsigned commits silently.

Signing mechanics are machine-specific and configured locally; the
rule for any agent is simply: always sign, never skip silently.
---

## GitHub PR Conventions

Every `gh pr create` call must include these minimum flags:

```bash
gh pr create --assignee J-MaFf --label <label> --body "Fixes #N\n\n<description>"
```

**Important details:**

- **Assignee:** Always `--assignee J-MaFf`

- **Labels:** Use GitHub default labels (e.g., `bug`, `enhancement`, `documentation`)

- **PR body must reference the issue:** Use `Fixes #N` format where N is the issue number. This auto-links and auto-closes the issue when merged.

- **Markdown formatting in `--body`:** When including code fences or formatted text in the PR body, use this pattern:
  ```bash
  gh pr create --body "$(cat <<'EOF'
  Fixes #123
  
  ## Changes
  - Updated auth logic
  
  ## Testing
  ```bash
  npm test
  ```
  EOF
  )"
  ```
  This ensures markdown renders correctly on GitHub.

- **Do NOT pass raw heredocs directly to `--body`.** Backslashes will render as literals in GitHub markdown. Always use the `"$(cat <<'EOF' ... EOF)"` wrapper above.

- **Keep PRs reviewable** — scope PRs to a single feature or bug fix, not a massive refactor. Large PRs are hard to review and more likely to introduce bugs. If a task feels too big, break it into multiple issues and PRs.

---

## Squash & Merge Commit Message Template

When merging a PR with squash and merge, **edit the commit message** to include the meaningful internal steps. Use this template:

```
<type>: <short description>

Fixes #<issue-number>

- <meaningful step 1>
- <meaningful step 2>
- <meaningful step 3>
```

**Example:**

```
feat: Add dark mode toggle

Fixes #42

- Extract color theme logic into ThemeProvider
- Add toggle button to header
- Update CSS for dark mode variants
- Add localStorage persistence for user preference
```

**Guidelines:**

- **First line:** Follow conventional commit format (`feat:`, `fix:`, `docs:`, etc.)
- **Issue reference:** Always include `Fixes #N` on its own line
- **Bullet points:** List only meaningful implementation steps; drop WIP commits and trivial changes
- **When to use:** Every squash merge — GitHub auto-generates a message, but replace it with this structure before confirming the merge

This keeps your commit history clean and linear while preserving the important context about what changed and why.

---

## PR Self-Review Checklist

Before marking a PR ready or asking for review, check for these common issues:

### PowerShell
- **No unused variables** — if a variable is assigned but never read, remove the assignment. In PowerShell, command exit status lives in `$LASTEXITCODE`, not in the captured output.
- **Resolve executables explicitly** — never hardcode `python`, `pip`, etc. On Windows, `python` may invoke the Microsoft Store stub or be absent. Use `Get-Command` to resolve in priority order: `py` → `python3` → `python` (Python Launcher first), `pip3` → `pip`.
- **Quoted paths for spaces in docs** — when documenting PowerShell commands with paths that contain spaces, use `& "path with spaces" -Args` syntax, not backtick-escape (`` .\path` with` spaces\script.ps1 ``). Backtick-escape is valid PowerShell but looks like a typo to readers unfamiliar with the language.

### Bash / shell scripts
- **Validate syntax before committing** — run `bash -n script.sh` to catch syntax errors.
- **Section ordering in git config INI files** — group settings logically: identity (`[user]`) first, then format (`[gpg]`), then behaviour (`[commit]`). Git ignores section order but readers expect it.

### Python
- **No silent side effects at import time** — don't call `subprocess` or install packages in the module-level `except ImportError` block. Raise a clear error and exit instead; let the install script handle dependencies.

### All languages
- **Post a self-review comment on every PR** — after opening a PR, re-read the diff as a reviewer would. If you find issues, fix them in a follow-up commit and document what you found and fixed in a PR comment. This creates a paper trail and catches problems before the user reviews.
- **Link follow-up issues back to the PR** — if the self-review surfaces something out of scope that you file as a separate GitHub issue (e.g. a pre-existing failure or unrelated bug), post a PR comment linking that issue. This keeps the paper trail connected so reviewers can see what was deferred and why.

---

## New Repository Setup

After creating a repo with `gh repo create`, apply these two settings before doing anything else.

### 1. Auto-delete head branches on merge + allow auto-merge

```bash
gh api repos/OWNER/REPO --method PATCH --field delete_branch_on_merge=true --field allow_auto_merge=true
```

GitHub enables neither by default. Without `delete_branch_on_merge`, merged branches pile up and `git cleanup` becomes necessary after every PR; it also powers stacked-PR retargeting (see Parallel Branches & Integration Waves). `allow_auto_merge` lets an approved merge fire as soon as required checks pass (`gh pr merge --auto`).

### 2. Protect `main` with a ruleset

Use a **repository ruleset** (not the legacy branch-protection API). Rulesets are the current GitHub mechanism and the standard for all repos. Create the "Main Branch Ruleset" via the rulesets API:

```bash
gh api repos/OWNER/REPO/rulesets \
  --method POST \
  --input - <<'EOF'
{
  "name": "Main Branch Ruleset",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "exclude": [],
      "include": [
        "refs/heads/main"
      ]
    }
  },
  "rules": [
    {
      "type": "non_fast_forward"
    },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 0,
        "dismiss_stale_reviews_on_push": true,
        "required_reviewers": [],
        "require_code_owner_review": false,
        "require_last_push_approval": false,
        "required_review_thread_resolution": true,
        "allowed_merge_methods": [
          "merge",
          "squash",
          "rebase"
        ]
      }
    }
  ],
  "bypass_actors": []
}
EOF
```

What this enforces on `main`:
- **`non_fast_forward`** — blocks force-pushes that rewrite history
- **`pull_request`** — all changes must land via PR; 0 required approvals (solo-repo friendly), stale reviews dismissed on push, and review threads must be resolved before merging
- All three merge methods stay allowed (merge / squash / rebase)

`bypass_actors: []` means no one bypasses the ruleset — not even admins. If CI status checks are added later, add a `required_status_checks` rule to the `rules` array.

> The response includes `id`, `source`, and `source_type` fields — those are instance-specific and are **not** part of the create payload; omit them when applying this template to a new repo.

---

## Project Documentation Files

Maintain a `STATUS.md` and `CHANGELOG.md` in every repo. Update them as part of the same PR that delivers the work — not as a separate afterthought.

### STATUS.md

Answers: *where is this project right now, and what would I do next?*

Required sections:
- **What This Is** — one paragraph, plain language
- **Current State — YYYY-MM-DD** — overall health (e.g. "all known issues resolved, `main` is clean") followed by:
  - **Components** table — file, description
  - **Resolved Issues** table — issue link, description, PR link
  - **Open Issues** — table or "None"
- **Natural Next Steps** — bulleted list of the most valuable things to do next
- **Prerequisites to Run** — minimum steps to get a working environment

Update the date and tables whenever a session closes with meaningful progress. This file is the first thing to read when resuming a project.

### CHANGELOG.md

Follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) format.

```markdown
## [Unreleased]

## [X.Y.Z] — YYYY-MM-DD
### Added
### Fixed
### Changed
### Removed
```

Rules:
- Keep an `[Unreleased]` section at the top; move its contents to a versioned entry on release
- Every PR that ships user-visible behavior gets an entry — one bullet per logical change
- Link the PR in each bullet: `([#N](url))`
- Use past tense, active voice: "Added X", "Fixed Y crashing when Z"
- Omit purely internal refactors that don't affect observable behavior

---

## Beads (`bd`) — Layered Task & Memory Workflow

**Applies only when a repo contains a `.beads/` directory.** Beads ([gastownhall/beads](https://github.com/gastownhall/beads)) is a Dolt-backed dependency-graph task tracker that sits **underneath** GitHub Issues as the agent's execution + memory layer. It does not replace this workflow — it layers under it.

### The layered model

- **GitHub Issue = the shippable unit.** Branch → PR → `Fixes #N` → squash-merge, exactly as above.
- **Beads = the execution/memory layer underneath.** Fine-grained tasks, a dependency graph, the `bd ready` queue, and `bd remember` persistent memory — the working notes beneath a shippable issue.

Beads guards **durability/sync**; these policies guard **review/control**. They only appear to conflict if you read bd's "push" as "merge." Keep them separate: bd makes work durable; **merges to `main` stay human-gated via PR** (never auto-merge).

### Core loop

```bash
bd ready                      # tasks with no open blockers
bd update <id> --claim        # claim one (sets assignee + in_progress)
# ...do the work...
bd close <id> --reason "..."  # close it
```

At **session close**, make work durable *without* merging:

```bash
git add <files> && git commit -S -m "..."   # signed, on the FEATURE branch
git push -u origin <feature-branch>          # push the branch, never main
bd dolt push                                 # sync the bead graph (refs/dolt/data)
```

Then open/update the PR and **stop at the merge gate** — a human approves the squash-merge.

### Cross-machine sync

Bead state syncs via **Dolt remotes** on the same git `origin`, under `refs/dolt/data` (separate from `refs/heads/*`). The Main Branch Ruleset targets `refs/heads/main` only, so `bd dolt push` is **not** blocked by branch protection.

- Fresh clone: `bd bootstrap` (clones the Dolt history the first time), then `bd dolt pull` thereafter.
- If `bd init` didn't auto-wire the remote: `bd dolt remote add origin git+https://github.com/<owner>/<repo>.git`, then `bd dolt push`. Commit the resulting `.beads/config.yaml` so fresh clones can bootstrap.

### `.beads/` commit convention — **dolt-only sync** (don't track the exports)

Bead state syncs **only** through Dolt (`refs/dolt/data`), not through git-tracked files. The JSONL
files are passive *exports* that bd rewrites to match the live DB on every mutation — tracking them
just produces endless git churn (stale exports, deletions stranded on `main`, "bd looks like it's
committing to `main`"). So **don't track them**:

- **Track:** `.beads/config.yaml` (carries `sync.remote`) and `.beads/metadata.json` (project
  identity) — these are stable and let a fresh clone `bd bootstrap`. Also `.beads/.gitignore` itself.
- **Ignore:** `.beads/embeddeddolt/` (local Dolt DB), `.beads/issues.jsonl`,
  `.beads/interactions.jsonl` (passive exports), `.beads/hooks/` (see below), runtime locks, and
  `.beads-credential-key`.

To convert a repo that currently tracks the exports:

```bash
git rm --cached .beads/issues.jsonl .beads/interactions.jsonl
git rm -r --cached .beads/hooks                       # if present and inert
# add issues.jsonl, interactions.jsonl, hooks/ to .beads/.gitignore
```

`issues.jsonl` is an **export for viewers/interchange, not the sync channel** — never `bd import` it
in place of `bd dolt pull`.

### Sync model (replaces "commit the export to a branch")

Because the exports aren't tracked, there's **no `.beads/` churn to commit** and no "branch before
every bd command" discipline to remember. Durability and cross-machine sync are pure Dolt:

```bash
bd dolt push     # at session close — sync the graph (refs/dolt/data), not blocked by branch protection
bd dolt pull     # pick up other machines' changes
bd bootstrap     # fresh clone: hydrate the local Dolt DB from the remote the first time
```

Still **never `git commit` `.beads/` changes onto `main` directly** — but with exports ignored the
only tracked `.beads/` files are `config.yaml` / `metadata.json`, which rarely change. If you *do*
see a diff on those that's **line-ending-only** (LF↔CRLF), it's noise — `git restore` it.

### Git hooks — opt-in, not tracked

bd can install git-hook shims (`pre-commit`, `post-merge`, `pre-push`, `post-checkout`,
`prepare-commit-msg`) that auto-sync Dolt and add agent trailers. They're **off by default**: tracking
them in `.beads/hooks/` without wiring them into `.git/hooks` just leaves inert files. Either:

- **Default (recommended for solo repos):** don't track them; sync explicitly with `bd dolt push` / `pull`.
- **Opt in per clone:** `bd hooks install` (writes into `.git/hooks`, local-only — doesn't travel via git).

Note the `prepare-commit-msg` hook rewrites commit messages with identity trailers and `pre-commit`
adds per-commit latency — fine for orchestrator fleets, usually unwanted noise for a solo repo.

### Memory

Use `bd remember "<insight>"` for **repo-scoped** knowledge that should travel with the repo. The **global** Claude memory system stays the home for cross-repo / user-level context — `bd remember` does not replace it.

### Caveats

- `bd init` appends a beads block to `CLAUDE.md` whose default wording mandates auto-push and bans `TaskCreate`/`MEMORY.md`. **Reconcile it** with these policies: push the feature branch + `bd dolt push`, keep the merge gated, and scope the memory rule to repo-level.
- After reconciling, `bd setup claude --check` reports the block as "stale." **Do not run `bd setup claude`** to clear it — that reverts the reconciliation.
- Install `bd` **checksum-verified**: the release binary checked against the release `checksums.txt`, or the official install script (which verifies for you).

---

## Summary: The Golden Rules

1. **Issue first, code second** — always create a GitHub issue before starting implementation
2. **Branch naming matters** — use `fix/`, `feat/`, or `docs/` prefixes
3. **Write detailed commit messages** — each commit should include relevant context and details for the final squash message
4. **Keep branches in sync with rebase** — use `git rebase main` to stay current; don't merge main into feature branches
5. **Force push safely** — use `git push --force-with-lease` after rebase, but only on feature branches, never on main
6. **Sign commits with SSH** — always use `git commit -S`; every machine signs with a file-based on-disk key (no agent, no Touch ID prompt); never skip silently
7. **Keep PRs reviewable** — scope PRs to a single feature or fix; break larger tasks into multiple PRs
8. **PRs reference issues** — every PR body must say `Fixes #N`
9. **Assignee and labels always** — every PR must have `--assignee J-MaFf` and an appropriate label
10. **Never auto-merge** — wait for explicit approval unless circumstances require it
11. **Correct markdown in PR bodies** — use the `"$(cat <<'EOF' ... EOF)"` pattern for formatted content
12. **Squash and merge with edited commit messages** — use the template to preserve meaningful steps in a clean, linear history
13. **Clean up merged branches** — use `git cleanup` alias after merging
14. **Self-review every PR** — re-read the diff after opening; fix issues in a follow-up commit and document findings in a PR comment
15. **Keep STATUS.md and CHANGELOG.md current** — update both as part of every PR that delivers work; STATUS.md shows where the project stands, CHANGELOG.md records what changed and links to the PR
16. **Configure every new repo on creation** — enable auto-delete branches and protect `main` before the first commit lands
17. **In beads repos, layer don't replace** — beads is the execution/memory layer; the GitHub Issue stays the shippable unit. Push the feature branch + `bd dolt push` at session close, but merges to `main` stay human-gated via PR (never auto-merge)
18. **Beads syncs via Dolt, not git** — don't track the JSONL exports (`issues.jsonl`/`interactions.jsonl`) or hook shims; gitignore them and sync with `bd dolt push` / `pull` (`bd bootstrap` on fresh clones). Only `config.yaml` + `metadata.json` stay tracked. Never commit `.beads/` to `main` directly
19. **Pick the integration pattern before opening parallel PRs** — independent changes: sequential bottom-up merge with rebase between (or a merge queue where the repo qualifies); dependent changes: a stacked chain merged bottom-up. Merge style never fixes conflicts — content overlap does
20. **Design away conflict-magnet files** — CHANGELOG `[Unreleased]` conflicts resolve as keep-both; committed generated files are resolved by regenerating from resolved sources, never hand-merged; prefer CI-published artifacts over committed generated files where distribution allows
