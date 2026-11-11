---
name: start
description: Start a new task by updating the default branch and creating a work branch named <type>/<short-description>. Use when the user wants to begin a task, feature, fix or any new piece of work.
argument-hint: "[type] <task description>"
allowed-tools: Bash(git *)
---

# Start a task

Create the branch the task will be developed on, always cut from an up-to-date default branch.

Task description from the user: $ARGUMENTS

## 1. Check the repository

- Run `git rev-parse --is-inside-work-tree`. If it is not a git repository, stop and tell the user.
- Find the default branch: `git symbolic-ref --short refs/remotes/origin/HEAD` (strip the `origin/` prefix). If that fails, use `main` if it exists, otherwise `master`. If the repository has no `origin` remote, work only with local branches and skip every fetch/pull below.
- Note the current branch with `git branch --show-current`.

## 2. Build the branch name

The name is `<type>/<slug>`.

- **type**: if the arguments start with a Conventional Commits type (`feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `perf`, `ci`, `build`, `style`), use it. Otherwise infer it from the description: new behaviour is `feat`, a bug is `fix`, maintenance is `chore`, and so on. When unsure, use `feat`.
- **slug**: a short summary of the description in the language the user wrote it, lowercase, accents removed, only `a-z`, `0-9` and `-`, at most 5 words and 40 characters. Example: "Adicionar login com o Google" → `feat/login-google`.
- If there is no description, ask the user what the task is before continuing.

## 3. Handle uncommitted changes

Run `git status --porcelain`. If there are changes, ask the user which option they want before doing anything:

1. **Bring them to the new branch** (common when work was started on the default branch by mistake)
2. **Stash them** and leave them out of the new branch
3. **Cancel**

For options 1 and 2, run `git stash push --include-untracked -m "flow:start <branch>"`.

## 4. Update the default branch

- If the current branch is not the default branch, tell the user the new branch will be cut from the default branch, not from the current one.
- `git fetch origin --prune`
- `git switch <default>`
- `git pull --ff-only`. If it fails because the local default branch has diverged from `origin`, stop and explain; never reset or force anything.

## 5. Create the branch

- If `<branch>` already exists locally (`git show-ref --verify refs/heads/<branch>`) or on the remote (`git ls-remote --heads origin <branch>`), ask whether to switch to the existing branch or choose another name.
- `git switch -c <branch>`
- If the user chose option 1 in step 3, run `git stash pop`. On conflict, stop and show the conflicting files.

Don't push the branch here; `/flow:commit` publishes it on the first commit.

## 6. Report

Tell the user in one or two lines the branch that was created, which branch it was cut from and its commit, and whether the stashed changes were brought along or are still stashed.

Don't start working on the task: don't read or change code, plan, or suggest an implementation. Stop after the report and wait for the user.
