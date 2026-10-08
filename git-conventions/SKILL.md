---
name: git-conventions
description: Shared git conventions the git-commit, git-push, git-pr, and git-rebase skills follow — how to find the default branch, which remote to push to, and which repos are personal. Not a workflow; read by the git skills so this lives in one place.
user-invocable: false
---

# Git conventions

- **Fork remote:** the user's fork remote is `powersj`. Push branches to the first remote that
  exists: `powersj`, then `fork`, then `origin`. `origin` is the upstream repo.
- **Personal repos** are ones whose `origin` is under the user's GitHub account
  (`github.com/powersj/`). Committing and pushing on the default branch is fine there.
  Every other repo needs a feature branch, even when `origin` is the only remote.
- **Default branch:** find it, don't assume `main`:

  ```sh
  git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null \
    || { git remote set-head origin --auto >/dev/null && git symbolic-ref --short refs/remotes/origin/HEAD; }
  ```

  The first command is instant when `origin/HEAD` is set, as it is in a clone. The fallback
  asks the remote once and saves the answer.
