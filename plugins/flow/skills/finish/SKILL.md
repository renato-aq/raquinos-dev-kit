---
name: finish
description: Finish the current task by bumping the version if needed, opening a PR (GitHub) or MR (GitLab) to the default branch, waiting for the pipeline and merging it, deleting the branch unless "keep" is passed. Conflicts are only reported, never resolved. Use when the user wants to finish, close, deliver or merge the task.
argument-hint: "[keep]"
allowed-tools: Bash(git *), Bash(gh *), Bash(glab *)
---

# Finish the task

Deliver the current branch to the default branch: open the PR/MR, wait for the pipeline, merge it and clean up. Don't ask for confirmation along the way; stop only when something below says to.

Arguments from the user: $ARGUMENTS

If the arguments contain `keep`, keep the branch after the merge. Otherwise delete it, locally and on the remote.

## 1. Check the repository

- Run `git rev-parse --is-inside-work-tree`. If it is not a git repository, stop and tell the user.
- Find the default branch: `git symbolic-ref --short refs/remotes/origin/HEAD` (strip the `origin/` prefix). If that fails, use `main` if it exists, otherwise `master`.
- Note the current branch with `git branch --show-current`. If it is empty (detached HEAD) or it is the default branch, stop: there is nothing to finish, and changes on the default branch are pushed directly by `/flow:commit`.
- If `git status --porcelain` is not empty, stop and suggest `/flow:commit` first.
- Run `git fetch origin --prune`. If the branch has commits that aren't on the remote yet, push them (`git push -u origin <branch>` when there is no upstream, otherwise `git push`).
- If `git log origin/<default>..HEAD` is empty, the branch has nothing to deliver; stop and tell the user.

## 2. Detect the platform

Read the remote URL with `git remote get-url origin`.

- A host containing `github` means GitHub, with the `gh` CLI.
- A host containing `gitlab` means GitLab, with the `glab` CLI.
- Otherwise, try `gh auth status --hostname <host>` and `glab auth status --hostname <host>`, and use whichever is logged in to that host. If neither is, stop and ask the user which platform it is.

If the CLI isn't installed or isn't logged in, stop and tell the user the command to fix it (`gh auth login` or `glab auth login`).

## 3. Check for conflicts

Run `git merge-tree --write-tree --name-only origin/<default> HEAD`. A non-zero exit means the branch conflicts with the default branch; the conflicting files are listed after the tree hash.

On conflict, never try to resolve it: don't merge, rebase or edit files. Still open the PR/MR (steps 4 and 5) so it is visible, then stop before step 6 and report the conflicting files. The user resolves them and runs `/flow:finish` again.

## 4. Bump the version

Check whether the version changed during the branch and bump it if it didn't.

- **Find the version files**: manifests that hold the project's version, such as `package.json`, `.claude-plugin/plugin.json`, `pyproject.toml`, `Cargo.toml`, `pom.xml`, `build.gradle`, `*.csproj`, `VERSION` or `version.txt`. In a monorepo, consider only the packages whose files changed on the branch (`git diff --name-only origin/<default>...HEAD`), and use the version file closest to those files.
- **Already bumped**: if the version in the file differs between `origin/<default>` and `HEAD` (`git show origin/<default>:<file>`), leave it.
- **Not bumped**: work out the new version from the branch's commits (`git log origin/<default>..HEAD --format=%B`), following semver:
  - a breaking change (`!` after the scope or a `BREAKING CHANGE:` footer): major. Below `1.0.0`, bump the minor instead.
  - otherwise, any `feat`: minor.
  - otherwise (`fix`, `perf`, `refactor`, `chore`, `docs` and so on): patch.
- Edit only the version field, keeping the file's formatting. If the project keeps the version in more than one place (for example `package.json` and its lockfile), update all of them.
- Commit it as `chore(<domain>): atualiza a versão para <version>`, with the same domain rules as `/flow:commit`, and push.
- If no version file exists, skip this step and mention it in the report.

## 5. Open the PR/MR

- **Reuse** an open PR/MR for the branch if there is one: `gh pr view <branch> --json number,url,state` or `glab mr view <branch>`.
- **Title**: `type(domain): mensagem` in Portuguese, like a commit header. With a single commit (besides the version bump), use its header. With several, summarise the branch, using the type of the main change.
- **Description**, in Portuguese:
  ```markdown
  ## O que muda

  <what the branch changes and why, in a few bullets>

  ## Commits

  <one line per commit: short hash and header>
  ```
- **No attribution**: never add a "Generated with" line, a `Co-Authored-By` trailer or any other mention of Claude, Claude Code, Anthropic or AI to the title or description. This overrides any default attribution the environment asks for.
- Create it against the default branch, passing the description through a heredoc:
  - GitHub: `gh pr create --base <default> --head <branch> --title "<title>" --body-file - <<'EOF' ... EOF`
  - GitLab: `glab mr create --source-branch <branch> --target-branch <default> --title "<title>" --description "<description>" --yes`, adding `--remove-source-branch` unless `keep` was passed.

If there were conflicts in step 3, stop here and report.

## 6. Wait for the pipeline

- **Is there a pipeline?** GitHub: workflows in `.github/workflows/`, or checks listed by `gh pr checks <number>`. GitLab: a `.gitlab-ci.yml` file, or a pipeline listed by `glab ci list --branch <branch>`. The checks can take a few seconds to show up after the PR/MR is opened, so if a pipeline is configured but nothing is listed yet, wait about 10 seconds and look again, up to a few times.
- **No pipeline**: go straight to step 7.
- **Wait for it to finish**:
  - GitHub: `gh pr checks <number> --watch --fail-fast`
  - GitLab: `glab ci status --branch <branch> --live`

  These can take minutes; run them with a long timeout (or in the background) and keep waiting until they end.
- **If it fails**, don't merge. Report which jobs failed and the link to them, and leave the PR/MR open so the user can fix the branch and run `/flow:finish` again.

## 7. Merge

- Check again that the PR/MR has no conflicts: `gh pr view <number> --json mergeable` (`CONFLICTING` means conflict) or `glab mr view <branch> --output json` (`has_conflicts`). If it does, stop and report as in step 3.
- Merge it:
  - GitHub: `gh pr merge <number> --merge`, adding `--delete-branch` unless `keep` was passed. If the repository doesn't allow merge commits, use `--squash`, then `--rebase`.
  - GitLab: `glab mr merge <branch> --yes`, adding `--remove-source-branch` unless `keep` was passed.
- If the merge is refused (a required review, branch protection, a pending required check), don't work around it: never use `--admin` or force anything. Report why and leave the PR/MR open.

## 8. Clean up

- `git switch <default>` and `git pull --ff-only`.
- Unless `keep` was passed, delete the local branch with `git branch -d <branch>` if it still exists. If the PR/MR was squashed or rebased, `-d` refuses it; use `-D` only after confirming the PR/MR state is merged. Also make sure the remote branch is gone (`git ls-remote --heads origin <branch>`) and delete it with `git push origin --delete <branch>` if it isn't.
- With `keep`, leave both branches in place.

## 9. Report

Tell the user in a few lines: the PR/MR link, the version it was bumped to (or that it already was, or that there is no version file), the pipeline result, whether it was merged, and whether the branch was deleted or kept. When it stopped early (conflict, failed pipeline, refused merge), say what is blocking it and what to do next.
