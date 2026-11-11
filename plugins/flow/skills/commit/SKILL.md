---
name: commit
description: Commit the current changes as Conventional Commits in the format type(domain): message, with the message in Portuguese, splitting the diff into one commit per domain and pushing after each commit. On the default branch, only simple changes (chore, docs, ci, build, style) are committed and pushed directly. Use when the user wants to commit, save or push their work.
argument-hint: "[hint about the change]"
allowed-tools: Bash(git *)
---

# Commit the changes

Turn the pending changes into one or more commits, grouped by domain, and push each one as soon as it is created. Don't ask for confirmation before committing; show what was done at the end.

Optional hint from the user about the change: $ARGUMENTS

## 1. Check the repository

- Run `git rev-parse --is-inside-work-tree`. If it is not a git repository, stop and tell the user.
- Note the current branch with `git branch --show-current`. If it is empty (detached HEAD), stop and tell the user.
- Find the default branch: `git symbolic-ref --short refs/remotes/origin/HEAD` (strip the `origin/` prefix). If that fails, use `main` if it exists, otherwise `master`. If the repository has no `origin` remote, commit only and skip every push below.
- If a merge, rebase or cherry-pick is in progress (`git status` says so), stop and tell the user to finish it first.

## 2. Collect the changes

- Run `git status --porcelain`. If it is empty, tell the user there is nothing to commit and stop.
- **What gets committed:**
  - If something is already staged (`git diff --staged --name-only` is not empty), commit only the staged changes; the user chose them. If any file is also in `git diff --name-only` (partially staged), make a single commit with everything staged and skip the split in step 3.
  - Otherwise, commit every change, including untracked files.
- Read the diff (`git diff`, `git diff --staged`) and the content of new files to understand what changed and why. Use the hint above when there is one.
- Leave out anything that looks like a secret or a local artifact (`.env*`, keys, credentials, build output, large binaries that aren't tracked yet) and tell the user at the end that it was left out.

## 3. Plan the commits

Split the changes into groups, one per domain, and write a message for each.

- **Domain**: the module, package or feature the files belong to, usually the most meaningful folder (in `plugins/flow/skills/commit/SKILL.md` the domain is `flow`; in `src/auth/login.ts` it is `auth`). Root files use their subject: `readme`, `deps`, `config`, `ci` and so on. Use lowercase kebab-case.
- **Type**: `feat` (new behaviour), `fix` (bug), `refactor`, `perf`, `test`, `docs`, `style` (formatting only), `build` (build system, dependencies), `ci`, `chore` (maintenance, version bumps). A group's type describes its main change.
- Keep a file whole in a single group, and keep files that only make sense together (a function and its test, a rename's old and new path) in the same group. If a group would mix unrelated types, split it further.
- Order the groups so each commit builds on the ones before it (shared code before what uses it).
- **Message**: `type(domain): mensagem`, in Portuguese, imperative mood, lowercase first letter, no trailing period, header at most 72 characters. Example: `feat(auth): adiciona login com o Google`. Add a body in Portuguese only when the reason for the change isn't obvious from the header. Add `!` after the scope and a `BREAKING CHANGE:` footer when the change breaks compatibility.

## 4. Check the branch

- **On a feature branch** (anything other than the default branch): continue to step 5.
- **On the default branch**: commit directly only when every planned commit is `chore`, `docs`, `ci`, `build` or `style`. If any commit would be another type, don't commit anything. Tell the user which changes look like `feat`/`fix`/etc. and suggest `/flow:start <description>`, which can bring the uncommitted changes to a new branch.

## 5. Commit and push each group

For each group, in order:

1. Stage only its files. When committing every change: `git add -A -- <paths>` (this also records deletions and renames). When committing only what was staged: first `git reset -q` once, before the first group, then `git add -A -- <paths>` for each group.
2. Commit with the planned message. Use a heredoc when there is a body:
   ```bash
   git commit -F - <<'EOF'
   type(domain): mensagem

   Corpo explicando o porquê.
   EOF
   ```
   Never use `--no-verify`. If a hook fails, show its output and stop. If the hook only reformatted files, stage them again and retry once.
3. Push:
   - If the branch has no upstream (`git rev-parse --abbrev-ref @{u}` fails), run `git push -u origin <branch>`. This is what publishes a branch created by `/flow:start`.
   - Otherwise, run `git push`.
   - If the push is rejected because the remote has new commits, run `git pull --rebase` and push again. If the rebase has conflicts, run `git rebase --abort`, stop and explain. Never force push.
   - If the push fails for any other reason, stop and show the error. The commits already made stay local.

## 6. Report

List the commits that were created (short hash and header), the branch they were pushed to, and anything that was left out or left uncommitted. On a feature branch, remind the user that `/flow:finish` opens the PR/MR when the task is done.
