---
name: git-rebase
description: "Rebase a feature branch onto the default branch (origin/main or origin/master), optionally squash it, and require confirmation before any force-push."
user-invocable: true
---

# Rebase

Before proceeding, inspect the branch and remotes with `git symbolic-ref -q --short HEAD` and `git remote`.

**Read the git conventions first** (`<base directory>/../git-conventions/SKILL.md`): it says how to find the default branch, which remote to push to, and which repos are personal.

Find the default branch as the git conventions show. Below, `<base>` is the result (for example
`origin/main`) and `<default>` is the branch name without `origin/`.

## Rules

- Rebase onto `<base>`, after `git fetch origin`.
- Ask before squashing; squashing discards per-commit history the user may want to keep.
- Ask before force-pushing, and use `--force-with-lease`, never bare `--force`.
- When squashing, regenerate the message from the combined diff rather than concatenating the old messages, and commit via the `git-commit` skill's message conventions (Linus style, `-s`, Co-Authored-By trailer, 72-col wrap, heredoc).
- Have the user review the regenerated message before applying it; the soft-reset and recommit rewrite history, so the message gets eyes before it lands.

## Workflow

1. Survey:

   ```sh
   git status
   git fetch origin
   git log --oneline <base>..HEAD
   ```

   If the working tree is dirty, stop and ask the user to commit (via `/git-commit`) or stash first. If on `<default>`, stop.

2. Rebase:

   ```sh
   git rebase <base>
   ```

   On conflicts, stop and report. Resolve only when the resolution is trivial and obvious; otherwise ask the user how to proceed.

3. Ask about squashing. Show `git log --oneline <base>..HEAD`, then:

   > Rebase complete — N commits ahead of `<base>`. Squash into one commit? Reply **yes** to squash, **no** to keep them separate.

   With a single commit there is nothing to squash; skip to step 5.

4. If squashing, regenerate the message:

   1. Read the combined diff and the existing messages for context:

      ```sh
      git diff <base>...HEAD
      git log <base>..HEAD
      ```

   2. Draft one new message from the combined diff: terse imperative subject, body explaining why the whole change is needed.
   3. Present it and ask:

      > Review the squashed commit message above. Reply **go** to apply, **edit** with revised text, or **stop** to cancel.

   4. On go, `git reset --soft <base>`, then commit the reviewed message via the `git-commit` skill's message conventions (`-s`, Co-Authored-By trailer, 72-col wrap, heredoc). The **go** reply is the approval; don't ask again.

5. Ask about force-pushing. A rebase rewrites history, so any prior push needs a force-push:

   > Force-push the rebased branch? Reply **yes** to push with `--force-with-lease`, **no** to leave it local.

   On yes, use the remote the `git-push` skill would pick (the git conventions' order):

   ```sh
   git push --force-with-lease <remote> <branch>
   ```

   On no, stop here.
