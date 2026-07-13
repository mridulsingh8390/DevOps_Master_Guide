# Git Complete Guide (Beginner to Advanced)
## For DevOps Engineers (Ubuntu/Linux, local/on-prem)

> This file is intentionally detailed.
> Each section includes:
> - What/Why
> - Commands
> - Explanation
> - Verification
> - Common mistakes and fixes
> - Practice tasks

---

## Table of Contents

1. What is Git and why it matters  
2. Installation and first-time setup  
3. Authentication (HTTPS vs SSH)  
4. Repository lifecycle  
5. Working tree, staging area, and commit lifecycle  
6. Branching and merging  
7. Rebase and history rewriting  
8. Undo and recovery  
9. Stash  
10. Diff, log, blame, bisect  
11. Tags and releases  
12. Remote management  
13. Gitignore and clean working directory  
14. Advanced useful commands  
15. Common failure scenarios  
16. Daily workflow cheat sheet  
17. Practice lab tasks

---

## 1) What is Git and Why it Matters

## What
Git is a distributed version control system that tracks file changes over time.

## Why
- Maintains complete history of your code
- Enables safe team collaboration
- Supports branching/parallel work
- Allows rollback and forensic debugging

## Key concept
Git has 3 major states:
1. **Working Directory** (your files on disk)
2. **Staging Area (Index)** (what will go in next commit)
3. **Repository (.git)** (commit history)

---

## 2) Installation and First-Time Setup

## Install
```bash
sudo apt update
sudo apt install -y git
git --version
```

## Global configuration
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global core.editor "vim"
git config --global color.ui auto
git config --global pull.rebase false
git config --global fetch.prune true
git config --global rerere.enabled true
```

## Show current config
```bash
git config --list --show-origin
```

## Why these settings?
- `init.defaultBranch main`: avoids legacy `master`
- `fetch.prune true`: cleans deleted remote tracking refs
- `rerere.enabled true`: remembers conflict resolutions

---

## 3) Authentication (HTTPS vs SSH)

## HTTPS
Simple but often requires PAT (Personal Access Token).

## SSH (recommended for developers)

### Generate key
```bash
ssh-keygen -t ed25519 -C "you@example.com"
```

### Start agent and add key
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

### Copy public key
```bash
cat ~/.ssh/id_ed25519.pub
```

Add this to Git provider (GitHub/GitLab/Bitbucket) SSH settings.

### Test
```bash
ssh -T git@github.com
```

## Optional SSH config
```bash
cat <<'EOF' >> ~/.ssh/config
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config
```

---

## 4) Repository Lifecycle

## Create new repo locally
```bash
mkdir myrepo && cd myrepo
git init
```

## Clone existing repo
```bash
git clone <repo_url>
git clone -b main <repo_url>
git clone --depth 1 <repo_url>
```

## Why `--depth 1`?
Shallow clone for faster CI pulls when full history isn’t needed.

## Inspect remote
```bash
git remote -v
git remote show origin
```

---

## 5) Working Tree, Staging Area, Commit Lifecycle

## Check status
```bash
git status
```

## Stage files
```bash
git add .
git add <file>
git add -p
git add -u
```

### Explanation
- `add .` stages all new/modified files
- `add -p` lets you stage selected hunks only
- `add -u` stages modified/deleted tracked files only

## Unstage files
```bash
git restore --staged <file>
```

## Commit
```bash
git commit -m "feat: add login handler"
git commit --amend
git commit --amend -m "feat: add login + validation"
```

## Verify history
```bash
git log --oneline --decorate -n 10
```

---

## 6) Branching and Merging

## Create/switch branch
```bash
git branch
git switch -c feature/auth
git checkout -b feature/auth
git switch main
```

## Merge branch
```bash
git merge feature/auth
git merge --no-ff feature/auth
```

## Why `--no-ff`?
Forces a merge commit to preserve branch context in history.

## Delete branch
```bash
git branch -d feature/auth
git branch -D feature/auth
```

---

## 7) Rebase and History Rewriting

## Rebase onto updated main
```bash
git switch feature/auth
git rebase main
```

## Interactive rebase
```bash
git rebase -i HEAD~5
```

You can:
- pick
- reword
- squash
- fixup
- drop

## Continue/abort
```bash
git rebase --continue
git rebase --abort
```

## Warning
Never rewrite history (`rebase`, `commit --amend`) on shared public branch unless team agrees.

---

## 8) Undo and Recovery

## Discard uncommitted file changes
```bash
git restore <file>
```

## Undo last commit but keep changes staged
```bash
git reset --soft HEAD~1
```

## Undo last commit and unstage changes
```bash
git reset --mixed HEAD~1
```

## Hard reset (danger)
```bash
git reset --hard HEAD~1
```

## Safe undo of a commit in shared branch
```bash
git revert <sha>
```

## Lifesaver: reflog
```bash
git reflog
```

Recover lost commit:
```bash
git checkout -b recovery <sha_from_reflog>
```

---

## 9) Stash

## Save temporary work
```bash
git stash
git stash push -m "WIP auth feature"
git stash push -u -m "WIP incl untracked"
```

## List/apply/pop
```bash
git stash list
git stash show -p stash@{0}
git stash apply stash@{0}
git stash pop
git stash drop stash@{0}
git stash clear
```

---

## 10) Diff, Log, Blame, Bisect

## Diff
```bash
git diff
git diff --staged
git diff main..feature/auth
```

## Logs
```bash
git log --oneline --graph --decorate --all
git log -p
git show <sha>
```

## Blame
```bash
git blame <file>
```

Shows who changed each line and when.

## Bisect (find bad commit)
```bash
git bisect start
git bisect bad
git bisect good <old_good_sha>
# test each step and mark:
git bisect good
git bisect bad
git bisect reset
```

---

## 11) Tags and Releases

## Annotated tag
```bash
git tag -a v1.0.0 -m "Release 1.0.0"
git tag
git show v1.0.0
```

## Push tags
```bash
git push origin v1.0.0
git push --tags
```

## Delete tag
```bash
git tag -d v1.0.0
git push origin :refs/tags/v1.0.0
```

---

## 12) Remote Management

## Add/change/remove remote
```bash
git remote add origin <repo_url>
git remote set-url origin <new_repo_url>
git remote rename origin upstream
git remote remove upstream
```

## Fetch/pull/push
```bash
git fetch
git fetch --all --prune
git pull
git pull --rebase
git push -u origin main
```

## Force push safer variant
```bash
git push --force-with-lease
```

`--force-with-lease` prevents overwriting others’ unseen updates.

---

## 13) .gitignore and Clean Working Directory

## Sample `.gitignore`
```gitignore
# Python
__pycache__/
*.pyc
.venv/

# Node
node_modules/

# IDE
.vscode/
.idea/

# OS
.DS_Store

# Secrets
.env
*.pem
```

## Clean untracked files (danger)
```bash
git clean -n
git clean -fd
```

- `-n`: dry-run preview
- `-fd`: force delete untracked files/dirs

---

## 14) Advanced Useful Commands

## Cherry-pick
```bash
git cherry-pick <sha>
git cherry-pick --continue
git cherry-pick --abort
```

## Submodules
```bash
git submodule add <repo_url> vendor/lib
git submodule update --init --recursive
```

## Garbage collection / integrity
```bash
git gc
git fsck
```

## Short aliases
```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.lg "log --oneline --graph --decorate --all"
```

---

## 15) Common Failure Scenarios

## Scenario 1: push rejected (non-fast-forward)
### Error
`Updates were rejected because the tip of your current branch is behind`

### Fix
```bash
git pull --rebase
git push
```

---

## Scenario 2: accidental hard reset
### Fix
```bash
git reflog
git checkout -b restore <sha>
```

---

## Scenario 3: detached HEAD confusion
### Fix
```bash
git switch -c temp-fix-branch
```

---

## Scenario 4: merge conflict
### Fix flow
1. open conflicted files  
2. resolve markers `<<<<<<< ======= >>>>>>>`  
3. stage  
```bash
git add <file>
git merge --continue   # or git rebase --continue
```

---

## 16) Daily Workflow Cheat Sheet

```bash
git switch main
git pull --rebase
git switch -c feature/new-task
# edit files
git add -p
git commit -m "feat: implement new-task"
git push -u origin feature/new-task
```

---

## 17) Practice Lab Tasks

1. Create repo and make 5 commits  
2. Create feature branch and merge via `--no-ff`  
3. Use `rebase -i` to squash commits  
4. Simulate bad commit and revert it  
5. Recover deleted commit using `reflog`  
6. Create and push annotated tag  
7. Use stash with untracked files and restore it  

---

## Final Notes

- Prefer `revert` on shared branches, `reset` for local cleanup.
- Use `--force-with-lease` instead of plain `--force`.
- Learn `reflog`; it saves hours.
