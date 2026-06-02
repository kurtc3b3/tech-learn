## What & When

**git** is distributed version control — commits, branches, merges, history. **GitHub CLI (`gh`)** talks to GitHub from the terminal — PRs, issues, Actions, repo settings — without leaving the shell.

Use when:

- **Day-to-day** source control on [[Python Development]] projects
- **Opening PRs** after [[Linting — pre-commit]] passes
- **Reviewing CI** status and merging from terminal
- **Cloning** templates ([[Python — Copier]]) and open-source repos

Overview: [[CLI]].

---

## Install

```bash
# macOS
brew install git gh

git --version
gh --version
gh auth login
```

Configure identity once:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

---

## git — Essential Workflow

```bash
# New branch
git checkout -b feature/add-endpoint
# or: git switch -c feature/add-endpoint

# Stage & commit
git status
git diff
git add path/to/file.py
git commit -m "Add health check endpoint"

# Sync with remote
git fetch origin
git pull origin main
git push -u origin HEAD
```

---

## git — Inspect History

```bash
git log --oneline -10
git log --oneline main..HEAD          # commits on branch only
git show HEAD
git blame src/app.py
git diff main...HEAD                  # full branch diff vs main
```

---

## git — Branch & Merge

```bash
git branch -a
git switch main
git pull origin main
git switch feature/add-endpoint
git merge main                        # bring main into feature
git rebase main                       # linear history (use carefully)
```

> [!warning] Never rebase shared main Avoid `git push --force` on `main`/`master` unless team policy explicitly allows it.

---

## git — Undo & Fix

```bash
git restore file.py                   # discard unstaged changes
git restore --staged file.py          # unstage
git commit --amend                    # fix last commit message (unpushed only)
git revert <commit-sha>               # safe undo on shared branches
git stash push -m "wip"               # shelve work
git stash pop
```

---

## git — Useful Flags

| Task | Command |
| --- | --- |
| Clone | `git clone https://github.com/org/repo.git` |
| Shallow clone | `git clone --depth 1 <url>` |
| Tag release | `git tag v1.2.0 && git push origin v1.2.0` |
| Clean untracked | `git clean -fd` (destructive — review first) |
| Remote URL | `git remote -v` |

---

## gh — Pull Requests

```bash
# From feature branch with commits pushed
gh pr create --title "Add health endpoint" --body "## Summary
- Adds GET /health

## Test plan
- [ ] pytest
"

gh pr list
gh pr view 42
gh pr checks 42
gh pr merge 42 --squash
gh pr checkout 42
```

Non-interactive body from file:

```bash
gh pr create --title "..." --body-file pr-body.md
```

---

## gh — Repo & Issues

```bash
gh repo clone org/repo
gh repo view --web
gh issue list
gh issue create --title "Bug: ..." --body "..."
gh workflow list
gh run list --workflow=ci.yml
gh run view <run-id> --log-failed
```

---

## gh — API & Scripting

```bash
gh api repos/{owner}/{repo}/pulls/1
gh api graphql -f query='
  query { viewer { login } }
'
```

JSON output for scripts — pair with [[Python — Typer]] admin tools if needed.

---

## Pre-commit Integration

[[Linting — pre-commit]] installs a git hook:

```bash
pre-commit install
git commit -m "..."   # hooks run automatically
```

Bypass only when intentional: `git commit --no-verify` (discouraged on shared repos).

---

## Typical PR Flow (This Vault)

```bash
git checkout -b docs/my-series
# edit notes...
git add Concepts/ Codes/
git commit -m "Add docs for X"
git push -u origin HEAD
gh pr create --title "..." --body "..."
gh pr merge --squash
git switch main && git pull origin main
```

---

## git vs gh

| Task | git | gh |
| --- | --- | --- |
| Local commits | ✅ | — |
| Push/fetch | ✅ | — |
| Open PR | — | ✅ |
| View CI | — | ✅ |
| Issues | — | ✅ |
| Merge on GitHub | — | ✅ |

---

## Quick Reference

```bash
git status && git diff
git add -p && git commit -m "message"
git push -u origin HEAD
gh pr create
gh pr checks
gh pr merge --squash
```

---

## Related Notes

- [[CLI]]
- [[Linting — pre-commit]]
- [[Python — Copier]]
- [[Python Development]]
- [[Commands/CLI — Docker & Compose]]
- [[Unit Testing - pytest]]

---

## Tags

#cli #git #github #gh #version-control #pr #devops
