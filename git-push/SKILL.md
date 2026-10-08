---
name: git-push
description: Push a branch to the user's fork remote (or fork, or origin), or a personal repo's default branch to origin, with explicit confirmation before any force-with-lease retry.
user-invocable: true
---

# Push

Before proceeding, inspect the branch and remotes with `git symbolic-ref -q --short HEAD` and `git remote`.

**Read the git conventions first** (`<base directory>/../git-conventions/SKILL.md`): it says how to find the default branch, which remote to push to, and which repos are personal.

Find the default branch as the git conventions show; `<default>` below is its name without
`origin/` (for example `main`). Check whether the repo is personal.

## Rules

- Refuse to push `<default>`, `main`, or `master`, unless the repo is personal: tell the user and stop. A personal repo's default branch goes to `origin`, and is **never force-pushed**.
- Force-push only after the user explicitly confirms, and only with `--force-with-lease` (never bare `--force`); the lease refuses the push if the remote moved in a way you haven't fetched, which protects a teammate's work.
- Target the first remote that exists, in the git conventions' order (the user's fork remote, then `fork`, then `origin`). A personal repo's default branch always goes to `origin`.
- Let hooks run: no `--no-verify` or other hook-skipping flags.

## Workflow

1. Check state with `git status -sb`. If the branch is `<default>`, `main`, or `master`, stop, unless the repo is personal.

2. Pick the remote, in the git conventions' order. For a personal repo's default branch, use `origin`.

3. Push:

   ```sh
   git push -u <remote> <branch>
   ```

   (`-u` is idempotent; a no-op if the upstream is already set.)

4. On a non-fast-forward rejection (`rejected — non-fast-forward` or similar):

   **On a personal repo's default branch, never offer a force-push.** Show the error, and
   suggest the commands in step 5 below so the user can pull and reconcile. On any other
   branch:

   1. Show the user the exact error output.
   2. Explain briefly: the remote has commits the local branch doesn't, usually because the branch was rebased or amended locally.
   3. Ask:

      > The remote rejected a fast-forward push. Force-push with `--force-with-lease`? Reply **yes** to force-push, **no** to stop and reconcile manually.

   4. On yes:

      ```sh
      git push --force-with-lease <remote> <branch>
      ```

   5. On no, suggest these so the user can see what diverged before deciding:

      ```sh
      git fetch <remote>
      git log <remote>/<branch>..HEAD
      git log HEAD..<remote>/<branch>
      ```
