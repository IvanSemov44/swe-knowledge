# Git

## What it is
A distributed version control system. Every developer has a full copy of the repository history. Changes are tracked as snapshots (commits), not diffs.

## Why it matters
You use Git every day. Knowing it well prevents losing work, enables clean collaboration, and lets you review history to understand why code was written a certain way.

---

## Core Concepts

**Repository (repo):** The project + its entire history.
**Working directory:** Your local files as you edit them.
**Staging area (index):** Files you've `git add`-ed, ready to be committed.
**Commit:** A snapshot of the staging area, with a message and parent pointer.
**Branch:** A lightweight pointer to a commit. Moving pointer, not a copy.
**HEAD:** Pointer to the currently checked-out branch (or commit in detached HEAD state).
**Remote:** Another copy of the repo (usually on GitHub/GitLab).

---

## Daily Commands

```bash
git status                     # what's changed, what's staged
git diff                       # unstaged changes
git diff --staged              # staged changes (what will be committed)

git add <file>                 # stage specific file
git add .                      # stage everything (careful with secrets)
git add -p                     # interactive staging (pick hunks)

git commit -m "message"        # commit staged changes
git commit --amend             # edit last commit (message or content)
                               # NEVER amend published commits

git log --oneline --graph      # pretty history
git log --oneline -10          # last 10 commits

git stash                      # save dirty working directory temporarily
git stash pop                  # restore stashed changes
git stash list                 # show all stashes
```

---

## Branching

```bash
git branch                     # list local branches
git branch -a                  # list all (including remote)

git checkout -b feature/xyz    # create and switch to new branch
git switch -c feature/xyz      # same (newer syntax)

git checkout main              # switch branch
git switch main                # same (newer)

git branch -d feature/xyz      # delete merged branch
git branch -D feature/xyz      # force delete unmerged branch
```

---

## Merging vs Rebasing

### Merge
```bash
git checkout main
git merge feature/xyz
```
Creates a **merge commit**. Preserves full history. Non-destructive.

```
main:    A → B → C → M
                     ↑
feature: A → B → D → E
```
M is the merge commit combining C and E.

**Fast-forward merge:** If main hasn't moved since you branched, git just moves the pointer forward (no merge commit).

### Rebase
```bash
git checkout feature/xyz
git rebase main
```
Replays feature commits on top of main. Creates **new commit hashes**. Linear history.

```
Before:  main: A → B → C
              feature: A → B → D → E

After:   main: A → B → C → D' → E'
              (D', E' are new commits with same changes)
```

**NEVER rebase a branch that others are working on.** It rewrites history and breaks their clones.

**Golden rule:**
- `merge` to incorporate public/shared branches (main → feature)
- `rebase` to clean up your own feature branch before a PR

### Interactive Rebase
```bash
git rebase -i HEAD~3    # rewrite last 3 commits
```
Options per commit: `pick`, `squash`, `fixup`, `reword`, `drop`

Use to: squash WIP commits, clean up message, reorder commits before PR.

---

## Remote Operations

```bash
git remote -v                          # show remotes
git remote add origin <url>            # add remote

git fetch origin                       # download remote changes (no merge)
git fetch origin main                  # fetch specific branch

git pull origin main                   # fetch + merge
git pull --rebase origin main          # fetch + rebase (cleaner)

git push origin feature/xyz            # push branch
git push -u origin feature/xyz         # push + set upstream (first time)
git push --force-with-lease            # safe force push (checks for upstream changes)
```

**`push --force` vs `--force-with-lease`:**
- `--force` overwrites remote unconditionally — dangerous on shared branches
- `--force-with-lease` fails if someone else pushed since you fetched — safer

---

## PR Workflow

1. Create branch: `git checkout -b feature/add-product`
2. Make changes, commit incrementally
3. Rebase on main before PR: `git rebase origin/main`
4. Clean up commits: `git rebase -i origin/main` (squash WIPs)
5. Push: `git push -u origin feature/add-product`
6. Open PR, fill out description, request review
7. After approval: squash merge or rebase merge to main
8. Delete branch

---

## Undoing Changes

```bash
# Undo unstaged changes in a file
git checkout -- <file>
git restore <file>             # newer syntax

# Unstage a file (keep changes)
git reset HEAD <file>
git restore --staged <file>    # newer syntax

# Undo last commit (keep changes staged)
git reset --soft HEAD~1

# Undo last commit (keep changes unstaged)
git reset HEAD~1

# Undo last commit (DISCARD changes — destructive)
git reset --hard HEAD~1

# Revert a commit (creates new commit, safe for shared branches)
git revert <commit-hash>

# View a specific commit's changes
git show <commit-hash>

# Find when a bug was introduced
git bisect start
git bisect bad         # current commit is bad
git bisect good <hash> # known good commit
# git runs binary search — mark each commit good/bad
```

---

## Cherry-pick

Apply a specific commit from another branch:
```bash
git cherry-pick <commit-hash>
```

Use case: Hotfix was committed to feature branch but needs to go to production immediately.

---

## Branching Strategies

### GitHub Flow (simple, recommended for most teams)
1. `main` is always deployable
2. Branch from `main` for every feature/fix
3. Open PR, review, merge
4. Deploy `main`

### GitFlow (complex, for scheduled releases)
- `main` = production
- `develop` = integration branch
- `feature/*` = feature branches off develop
- `release/*` = release preparation
- `hotfix/*` = urgent production fixes

### Trunk-Based Development
- Everyone commits to `main` daily
- Feature flags to hide incomplete features
- Very short-lived branches (< 1 day)
- Requires strong CI/CD and test coverage

---

## .gitignore

```gitignore
# .NET
bin/
obj/
*.user
.vs/
appsettings.Development.json  # never commit secrets

# Node
node_modules/
dist/
.env
.env.local

# IDEs
.idea/
*.swp
```

---

## Common Interview Questions

1. What is the difference between `git fetch` and `git pull`?
2. When would you use rebase vs merge?
3. What does `git reset --hard` do? When is it dangerous?
4. How do you undo a commit that has already been pushed?
5. What is a detached HEAD state?
6. What is the difference between `git stash` and a commit?

---

## Common Mistakes

- Committing secrets (API keys, passwords) — even if you delete them in the next commit, history is permanent
- `git add .` without checking `git status` first
- `git reset --hard` without realizing it discards uncommitted changes
- Force-pushing to `main` / shared branches
- Rebasing public/shared branches (rewrites history others depend on)
- Writing commit messages like "fix" or "wip" with no context

---

## How It Connects

- GitHub Actions CI/CD triggers on push/PR to specific branches
- Feature flags + trunk-based development enables continuous deployment
- Semantic versioning + Git tags mark release points
- `git bisect` is a practical application of binary search O(log n)

---

## My Confidence Level
- `[c]` Add, commit, push, pull
- `[c]` Branching, merging
- `[b]` Rebase vs merge
- `[b]` PR workflow
- `[~]` Cherry-pick, stash
- `[ ]` GitFlow vs trunk-based strategies

## My Notes
<!-- Personal notes -->
