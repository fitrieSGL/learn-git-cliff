## Gitflow + git-cliff: Full Process

## Branch Structure

```
master
release/v1.0.0
hotfix/bug
develop
feature/add-homepage
```

---

## Setup (one-time)

```bash
# install git-cliff
brew install git-cliff        # macOS
cargo install git-cliff       # via Rust

# init config in your repo root
git cliff --init
```

Edit `cliff.toml` to skip release chore commits:

```toml
[git]
commit_parsers = [
  { message = "^chore: release", skip = true },
  { message = "^chore: update changelog", skip = true },
  { message = "^feat", group = "Features" },
  { message = "^fix", group = "Bug Fixes" },
]
```

---

## Conventional Commit Format

git-cliff reads your commit messages to build the changelog. Always use this format:

```
fix: resolve null crash on login       → patch bump (1.0.0 → 1.0.1)
feat: add homepage                     → minor bump (1.0.0 → 1.1.0)
feat!: redesign auth API               → major bump (1.0.0 → 2.0.0)
chore: release v1.0.1                  → skipped (ignored by cliff.toml)
```

---

## Scenario 1 — Feature → Develop

> Features never touch master directly. No changelog here.

```bash
# cut feature branch from develop
git checkout develop
git checkout -b feature/add-homepage

# work and commit
git commit -m "feat: add homepage layout"
git commit -m "feat: add homepage hero section"

# merge back to develop
git checkout develop
git merge --no-ff feature/add-homepage
git push origin develop

# clean up
git branch -d feature/add-homepage
```

---

## Scenario 2 — Release → Master

> Changelog is written on the release branch, tag goes on master.

```bash
# cut release branch from develop
git checkout develop
git checkout -b release/v1.0.0

# generate changelog + bump version on the release branch
git cliff --tag v1.0.0 -o CHANGELOG.md
git add CHANGELOG.md
git commit -m "chore: release v1.0.0"

# merge to master with --no-ff (always)
git checkout master
git merge --no-ff release/v1.0.0

# tag on master (not on release branch)
git tag v1.0.0
git push --follow-tags

# sync back to develop (important — keeps develop in sync)
git checkout develop
git merge --no-ff release/v1.0.0
git push origin develop

# clean up
git branch -d release/v1.0.0
```

---

## Scenario 3 — Hotfix → Master

> Hotfix is cut from master, not develop. Changelog uses --unreleased flag.

```bash
# cut hotfix from master
git checkout master
git checkout -b hotfix/bug

# fix and commit
git commit -m "fix: resolve null crash on checkout"

# generate changelog (only unreleased commits since last tag)
git cliff --tag v1.0.1 --unreleased --prepend CHANGELOG.md
git add CHANGELOG.md
git commit -m "chore: release v1.0.1"

# merge to master with --no-ff (always)
git checkout master
git merge --no-ff hotfix/bug

# tag on master
git tag v1.0.1
git push --follow-tags

# sync back to develop (keeps the fix in develop too)
git checkout develop
git merge --no-ff hotfix/bug
git push origin develop

# clean up
git branch -d hotfix/bug
```

---

## Why --no-ff?

`--no-ff` forces a merge commit even when git could fast-forward. Always use it for Gitflow merges.

```bash
# without --no-ff (fast-forward) — branch history lost
* def5678 fix: resolve null crash
* jkl3456 chore: release v1.0.0

# with --no-ff — branch shape preserved
*   abc1234 Merge branch 'hotfix/bug'
|\
| * def5678 fix: resolve null crash
|/
* jkl3456 chore: release v1.0.0
```

| Merge | Flag |
|---|---|
| `feature → develop` | `--no-ff` or `--squash` (team preference) |
| `release → master` | `--no-ff` always |
| `hotfix → master` | `--no-ff` always |
| `hotfix → develop` | `--no-ff` always |
| `release → develop` | `--no-ff` always |

---

## Tag Rules

| Branch | Tag? |
|---|---|
| `feature/*` | Never |
| `develop` | Never |
| `release/*` | Never — tag on master after merge |
| `hotfix/*` | Never — tag on master after merge |
| `master` | Always after every merge |

> The tag marks what is live in production. Only master is production.

---

## git-cliff Flags Reference

| Flag | What it does |
|---|---|
| `--tag v1.0.1` | sets the version for the changelog header |
| `--unreleased` | only commits since the last tag (use for hotfixes) |
| `--prepend CHANGELOG.md` | adds new section to top of existing file |
| `-o CHANGELOG.md` | writes full changelog (overwrites) |
| `--latest` | only the most recent tag's commits |

---

## Squash Merge Note

If you use `git merge --squash hotfix/bug`, git-cliff reads the **squash commit message** on master — not the individual commits on the hotfix branch. Make sure the squash commit message follows conventional format:

```bash
git merge --squash hotfix/bug
git commit -m "fix: resolve null crash on checkout"   # must be conventional format
git cliff --tag v1.0.1 --unreleased --prepend CHANGELOG.md
```

---

## Recommended cliff.toml (Full)

```toml
[changelog]
header = "# Changelog\n\n"
body = """
{% for group, commits in commits | group_by(attribute="group") %}
### {{ group | upper_first }}
{% for commit in commits %}
- {{ commit.message | upper_first }} ([{{ commit.id | truncate(length=7, end="") }}]({{ commit.id }}))
{% endfor %}
{% endfor %}
"""
trim = true

[git]
conventional_commits = true
filter_unconventional = true
commit_parsers = [
  { message = "^chore: release", skip = true },
  { message = "^chore: update changelog", skip = true },
  { message = "^chore", skip = true },
  { message = "^feat", group = "Features" },
  { message = "^fix", group = "Bug Fixes" },
  { message = "^docs", group = "Documentation" },
  { message = "^refactor", group = "Refactor" },
]
```
