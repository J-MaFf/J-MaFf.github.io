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
   - Example: `gh issue create --title "Fix auth timeout" --label bug`

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

### 1. Auto-delete head branches on merge

```bash
gh api repos/OWNER/REPO --method PATCH --field delete_branch_on_merge=true
```

GitHub does not enable this by default. Without it, merged branches pile up and `git cleanup` becomes necessary after every PR.

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

## Summary: The Golden Rules

1. **Issue first, code second** — always create a GitHub issue before starting implementation
2. **Branch naming matters** — use `fix/`, `feat/`, or `docs/` prefixes
3. **Write detailed commit messages** — each commit should include relevant context and details for the final squash message
4. **Keep branches in sync with rebase** — use `git rebase main` to stay current; don't merge main into feature branches
5. **Force push safely** — use `git push --force-with-lease` after rebase, but only on feature branches, never on main
6. **Sign commits with SSH** — always use `git commit -S`; never skip silently
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
