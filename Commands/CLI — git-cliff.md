## What & When

**git-cliff** generates **changelogs and release notes** from Git history — grouped by Conventional Commits, scoped to version tags. Best paired with **squash merge** PRs on GitHub so `main` has one clean commit per change.

Use git-cliff when:

- You tag releases (`v1.0.0`, `v1.1.0`) and want **CHANGELOG.md** or GitHub Release bodies
- PR titles follow **Conventional Commits** (`feat:`, `fix:`, `docs:`)
- You automate releases with **GitHub Actions** on tag push
- You want issue → PR → release traceability without hand-writing notes

```bash
brew install git-cliff          # macOS
# or: cargo install git-cliff
```

Overview: [[CLI]]. Git workflow: [[Commands/CLI — Git & GitHub]].

---

## git-cliff vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Changelog from git history | **git-cliff** | `cliff.toml` + tags |
| Git / PRs / releases | [[Commands/CLI — Git & GitHub]] | `gh release create` |
| Enforce commit message format | commitlint / pre-commit `commit-msg` | [[Linting — pre-commit]] |
| CI on tag push | GitHub Actions | See below |

---

## Pipeline

```text
Issue → PR (Conventional title) → Squash merge → main
    → git tag v1.2.0 → git push --tags
    → git-cliff --latest → CHANGELOG / GitHub Release
```

---

## Install & Init

```bash
git-cliff --version

# In repo root
git-cliff --init          # creates cliff.toml
git add cliff.toml
git commit -m "chore: add git-cliff configuration"
```

---

## Conventional Commits (Required)

**Good** (squash commit / PR title):

```text
feat(auth): add oauth login
fix(api): handle null response
docs(readme): update installation
perf(search): improve indexing
feat(api)!: remove legacy endpoint    # breaking
```

**Avoid:** `fixed stuff`, `wip`, `update`

| Type | Meaning |
| --- | --- |
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation |
| `perf` | Performance |
| `refactor` | Restructure, no behavior change |
| `test` | Tests |
| `ci` | CI/CD |
| `chore` | Maintenance |

Breaking changes: `feat(scope)!:` or footer `BREAKING CHANGE:`.

---

## Squash Merge vs Other Strategies

| Merge style | What git-cliff reads | What matters |
| --- | --- | --- |
| **Squash merge** (recommended) | One commit per PR on `main` | **PR title** (becomes squash message) |
| Merge commit | All commits on branch | Every commit message |
| Rebase merge | Every rebased commit | Every commit message |

**Recommendation:** GitHub → Settings → allow **Squash merge only**; use Conventional PR titles:

```text
feat(auth): add oauth login (#123)
fix(api): handle timeout retry (#124)
```

Branch commits can be messy (`wip`, `fix test`); the squash title is the release record.

---

## Generate Changelog

```bash
git-cliff                      # preview to stdout
git-cliff > CHANGELOG.md       # full history
git-cliff --latest > CHANGELOG.md   # since last tag
git-cliff --unreleased         # changes after latest tag
```

Tag releases with semver:

```bash
git tag v1.2.0
git push origin v1.2.0
```

Example output:

```markdown
## v1.2.0 - 2026-06-05

### Features
- Add OAuth login

### Bug Fixes
- Handle timeout retry
```

---

## `cliff.toml` (Squash + Conventional Commits)

```toml
[changelog]
header = """
# Changelog

All notable changes to this project will be documented in this file.
"""

body = """
{% if version %}
## {{ version }} - {{ timestamp | date(format="%Y-%m-%d") }}
{% else %}
## Unreleased
{% endif %}

{% for group, commits in commits | group_by(attribute="group") %}
### {{ group }}

{% for commit in commits %}
- {{ commit.message | upper_first }}
{% endfor %}
{% endfor %}
"""

trim = true

[git]
conventional_commits = true
filter_unconventional = true
split_commits = false
tag_pattern = "v[0-9].*"

commit_parsers = [
  { message = "^feat", group = "Features" },
  { message = "^fix", group = "Bug Fixes" },
  { message = "^perf", group = "Performance" },
  { message = "^refactor", group = "Refactoring" },
  { message = "^docs", group = "Documentation" },
  { message = "^test", group = "Testing" },
  { message = "^build", group = "Build System" },
  { message = "^ci", group = "CI/CD" },
  { message = "^chore", group = "Maintenance" },
]

protect_breaking_commits = true
```

Keep config small — quality comes from **PR titles**, not complex templates.

---

## GitHub Actions (Tag → Release Notes)

```yaml
# .github/workflows/release.yml
name: Release Notes

on:
  push:
    tags:
      - "v*"

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install git-cliff
        run: |
          curl -sSL \
            https://github.com/orhun/git-cliff/releases/latest/download/git-cliff-x86_64-unknown-linux-gnu.tar.gz \
            | tar -xz

      - name: Generate changelog
        run: ./git-cliff --latest > RELEASE_NOTES.md

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          body_path: RELEASE_NOTES.md
```

Manual release flow:

```bash
git tag v1.3.0 && git push origin v1.3.0
git-cliff --latest > RELEASE_NOTES.md
gh release create v1.3.0 --notes-file RELEASE_NOTES.md
```

---

## Team Standards (Minimal)

| Rule | Example |
| --- | --- |
| Branch names | `feature/add-oauth`, `fix/token-refresh` |
| PR title | `feat(auth): add oauth login` |
| PR body | `Closes #123` |
| Merge | Squash only |
| Tags | `v1.0.0`, `v1.1.0` |
| Generate | `git-cliff --latest` |

Optional: **commitlint** on `commit-msg` pre-commit hook if you use rebase merge or require local commit format — [[Linting — pre-commit]].

---

## Quick Reference

| Task | Command |
| --- | --- |
| Init config | `git-cliff --init` |
| Preview | `git-cliff` |
| Since last tag | `git-cliff --latest` |
| Write CHANGELOG | `git-cliff --latest > CHANGELOG.md` |
| Tag release | `git tag v1.2.0 && git push origin v1.2.0` |
| Config file | `cliff.toml` |

---

## Related Notes

- [[CLI]]
- [[Commands/CLI — Git & GitHub]]
- [[Linting — pre-commit]]
- [[Python Development]]

---

## Tags

#cli #git-cliff #changelog #conventional-commits #github #releases #devops #semver
