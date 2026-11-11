---
name: push
description: Push the current branch to the remote as work in progress, without opening a PR/MR. If there are uncommitted changes, commits them first with /flow:commit. Use when the user wants to push, sync, back up or send the current state of the branch to the repository.
argument-hint: "[hint about the change]"
allowed-tools: Bash(git *)
---

# Push the branch

Send the current state of the branch to the remote, as work in progress. This only pushes: it doesn't open a PR/MR, bump the version or merge anything; that is `/flow:finish`. Don't ask for confirmation; show what was done at the end.

Optional hint from the user about the change: $ARGUMENTS

## 1. Check the repository

- Run `git rev-parse --is-inside-work-tree`. If it is not a git repository, stop and tell the user.
- Note the current branch with `git branch --show-current`. If it is empty (detached HEAD), stop and tell the user.
- If the repository has no `origin` remote (`git remote get-url origin` fails), stop: there is nowhere to push.
- Find the default branch: `git symbolic-ref --short refs/remotes/origin/HEAD` (strip the `origin/` prefix). If that fails, use `main` if it exists, otherwise `master`.
- If a merge, rebase or cherry-pick is in progress (`git status` says so), stop and tell the user to finish it first.

## 2. Commit pending changes

If `git status --porcelain` is not empty, the changes haven't been committed yet. Run `/flow:commit` first, invoking the `flow:commit` skill with the Skill tool and passing the hint above as its arguments. It splits the changes into commits and pushes each one.

- If it stops without committing (a failing hook, a `feat`/`fix` on the default branch, a push error), stop here too and report why.
- If it leaves files out (secrets, local artifacts), don't commit them here; mention them in the report.

## 3. Push the remaining commits

- Run `git fetch origin`.
- **No upstream** (`git rev-parse --abbrev-ref @{u}` fails): if the branch has no commits of its own yet (`git log origin/<default>..HEAD` is empty), tell the user there is nothing to push and stop. Otherwise, on the default branch apply the check below first, then run `git push -u origin <branch>`.
- **With an upstream**: list the commits not on the remote with `git log @{u}..HEAD --oneline`. If it is empty, the branch is already up to date; say so and go to the report.
- **On the default branch**: push only when every unpushed commit is `chore`, `docs`, `ci`, `build` or `style`, as in `/flow:commit`. If any is another type, don't push. List those commits and suggest `/flow:start <description>` to move the work to a branch.
- Run `git push`.
  - If it is rejected because the remote has new commits, run `git pull --rebase` and push again. If the rebase has conflicts, run `git rebase --abort`, stop and explain.
  - Never force push. If the remote branch was rewritten and the branches have diverged, stop and explain.
  - If the push fails for any other reason, stop and show the error. The commits stay local.

## 4. Report

In a few lines: the commits created by `/flow:commit` (if any), the commits pushed (short hash and header), the branch and remote they went to, and anything left out or uncommitted. On a feature branch, remind the user that `/flow:finish` opens the PR/MR when the task is done.
