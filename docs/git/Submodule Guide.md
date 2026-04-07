# Git Submodule Guide

Reference for working with the notiguide monorepo submodule setup.

## Repository layout

```
notiguide/                    (master repo — github.com/Thomas-Hoang-04/notiguide-master)
├── backend/                  (submodule — notiguide-be.git)
├── web/                      (submodule — notiguide-admin.git)
├── client-web/               (submodule — notiguide-client.git)
├── transmitter/              (submodule — notiguide-transmitter.git)
├── docs/                     (tracked directly in master repo)
├── CLAUDE.md                 (tracked directly in master repo)
└── .gitmodules               (submodule registry)
```

Each submodule has its own independent git history, branches, and remote. The master repo only stores a **pointer** (commit SHA) to each submodule — not its contents.

---

## Daily workflow

### 1. Pull latest everything (fresh clone or sync)

```bash
# Clone master with all submodules
git clone --recurse-submodules <master-repo-url>

# Or if already cloned without submodules
git submodule update --init --recursive
```

### 2. Pull updates across all submodules

```bash
# From master repo root
git submodule update --remote --merge
```

This fetches each submodule's latest `main` (or tracked branch) and merges it into your local checkout.

To pull just one:

```bash
git submodule update --remote --merge backend
```

### 3. Work inside a submodule

```bash
cd backend

# You're now in a full git repo — normal git workflow
git checkout main
git pull
# ... make changes ...
git add -A
git commit -m "feat: add something"
git push
```

**Important**: After committing and pushing inside a submodule, go back to the master repo and update the pointer:

```bash
cd ..   # back to master root
git add backend
git commit -m "chore: update backend submodule pointer"
git push
```

### 4. Check submodule status

```bash
# See which commit each submodule points to
git submodule status

# See if any submodule has local changes
git submodule foreach 'git status --short'

# See unpushed commits in all submodules
git submodule foreach 'git log --oneline origin/main..HEAD 2>/dev/null'
```

---

## Updating submodule pointers

The master repo records a specific commit SHA for each submodule. When you (or a collaborator) push new commits to a submodule's remote, the master repo doesn't automatically update.

```bash
# Update all submodules to their latest remote commit
git submodule update --remote

# Stage the updated pointers
git add backend web client-web receiver
git commit -m "chore: update submodule pointers"
git push
```

---

## Branching across submodules

Submodule branches are independent. If you need a feature branch that spans multiple repos:

```bash
# Create branch in each relevant submodule
cd backend && git checkout -b feat/new-feature
cd ../web && git checkout -b feat/new-feature

# Work, commit, push in each
cd backend && git push -u origin feat/new-feature
cd ../web && git push -u origin feat/new-feature

# In master repo, the pointer tracks whatever commit is checked out
cd ..
git add backend web
git commit -m "chore: point to feat/new-feature branches"
```

When done, merge each submodule's branch independently, then update the master pointers.

---

## Common operations

### Add a new submodule

```bash
git submodule add https://github.com/user/repo.git path/to/dir
git commit -m "chore: add new-repo submodule"
```

### Remove a submodule

```bash
# 1. Deinit
git submodule deinit -f <path>

# 2. Remove from .gitmodules and index
git rm -f <path>

# 3. Clean up .git/modules cache
rm -rf .git/modules/<path>

# 4. Commit
git commit -m "chore: remove <path> submodule"
```

### Change a submodule's tracked branch

By default, `git submodule update --remote` tracks `main`. To change:

```bash
# Edit .gitmodules
git config -f .gitmodules submodule.backend.branch develop

# Then update
git submodule update --remote backend
```

### Reset a submodule to the pointer commit

If a submodule is in a detached HEAD or dirty state and you want to reset:

```bash
git submodule update --force --checkout backend
```

---

## Gotchas

### Detached HEAD after `git submodule update`

`git submodule update` (without `--remote`) checks out the exact commit the master repo points to, which results in a **detached HEAD**. To get back on a branch:

```bash
cd backend
git checkout main
```

### Forgetting to push the submodule before pushing master

If you commit a new submodule pointer in the master repo but haven't pushed the submodule itself, collaborators will get an error when they try to update. Always:

1. Push the submodule first
2. Then update and push the master pointer

### Collaborator cloned without `--recurse-submodules`

They'll see empty submodule directories. Fix:

```bash
git submodule update --init --recursive
```

### Submodule shows `(modified content)` in master `git status`

This means the submodule has uncommitted changes or its checked-out commit differs from what master expects. Either:

- Commit/discard changes inside the submodule, or
- Update the master pointer: `git add <submodule> && git commit`

---

## Quick reference

| Task | Command |
|---|---|
| Clone with submodules | `git clone --recurse-submodules <url>` |
| Init submodules after clone | `git submodule update --init --recursive` |
| Pull all submodule updates | `git submodule update --remote --merge` |
| Check status | `git submodule status` |
| Update master pointer | `git add <submodule> && git commit` |
| Run command in all submodules | `git submodule foreach '<command>'` |
| See diff of pointer changes | `git diff --submodule` |
