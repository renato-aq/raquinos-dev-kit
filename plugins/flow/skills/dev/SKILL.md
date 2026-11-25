---
name: dev
description: Implement the specs written by /flow:spec in .raquinos-flow/specs/, one subagent per spec, either one after the other or in parallel batches where each spec runs in its own git worktree. Asks the user which mode to use, has every spec reviewed against the spec, the ADRs and the architecture before concluding it, commits each spec and records what was done. Use when the user wants to implement, develop, build or run the specs of the current task.
argument-hint: "[spec numbers, e.g. 002 or 000-003]"
allowed-tools: Agent, Skill, Read, Grep, Glob, Edit(.raquinos-flow/**), Bash(git *)
---

# Implement the specs

Implement the specs of the current task. This session only orchestrates: it reads the specs, plans the order, launches the subagents, keeps the spec files up to date and commits. **All implementation and all review happen in subagents, never in this session**: don't write or change project code here, not even for a small fix; send it to a subagent.

Every subagent is launched with the Agent tool using `subagent_type: "general-purpose"`, with a fixed model and effort, whatever the size of the work:

- **Implementing and fixing**: `model: "opus"` (Opus 5.5) and `effort: "medium"`.
- **Reviewing**: `model: "fable"` (Fable 5.2) and `effort: "high"`.
 If the Agent tool isn't available, stop and tell the user; don't implement here instead.

This doesn't write tests (a separate skill does that) and doesn't open a PR/MR.

Arguments from the user: $ARGUMENTS

## 1. Load the specs

- List `.raquinos-flow/specs/` at the repository root and read every spec. If the directory doesn't exist or is empty, stop and suggest `/flow:spec <task>` first.
- **Select the specs to implement**: the ones whose status is `pendente` or `em andamento`. If the arguments name spec numbers (`002`, `000 003`, `000-003`), restrict the selection to those. If nothing is left to implement, say that every spec is `concluida`, suggest `/flow:finish`, and stop.
- A selected spec that depends on a spec that is neither `concluida` nor selected can't run. Tell the user which one and why, and leave it out.
- A spec already `em andamento` was started by an earlier run. Keep it selected and tell its subagent that part of it may already be implemented.

Then check the repository, because each spec ends in a commit:

- Find the current branch with `git branch --show-current` and the default branch with `git symbolic-ref --short refs/remotes/origin/HEAD` (strip `origin/`; fall back to `main`, then `master`). On the default branch or with a detached HEAD, stop and suggest `/flow:start <description>`: specs are implemented on a work branch.
- If `git status --porcelain` is not empty, stop and suggest `/flow:commit` first. The tree must be clean so that each spec's commit holds only that spec's work.
- If a merge, rebase or cherry-pick is in progress, stop and tell the user to finish it first.
- If `git worktree list` or `git branch --list 'flow-dev/*'` shows worktrees or branches left by an earlier run, show them to the user and ask whether to discard them before going on. Never remove them without that answer; they may hold work that wasn't integrated.

## 2. Gather the shared context

Collect once what every subagent needs, so each brief is self-contained:

- The ADRs of the project (look in `docs/adr/`, `docs/adrs/`, `docs/decisions/`, `docs/architecture/decisions/`, `doc/adr/`, `adr/`): their **absolute** paths, and a few lines on the architecture they define. With no ADRs, describe in a few lines the architecture the code already follows.
- The stack, the folder layout and the conventions that matter for this task.
- For each selected spec, the files and modules it will create or change, taken from the spec's notes and from a look at the code. This is what step 3 uses to find collisions, so be concrete: paths, not "the auth module".

## 3. Plan the execution

Work out which specs can run at the same time. Two specs can run in parallel only when both are true:

- **No dependency** between them, direct or through other specs (`Depende de`).
- **No collision**: they don't create or change the same file, and neither needs something the other one creates (a type, a function, a table, a route, a configuration key). Shared registration files (routes, dependency injection, exports, migrations, lockfiles) count as collisions. When in doubt, treat it as a collision.

Each parallel spec runs in its own worktree, so two agents never write to the same directory. The collision rule still matters: it is what lets their work be joined afterwards without conflicts.

Group the selected specs into **batches**: every spec in a batch can run in parallel with the others in it, and a batch only starts after the previous ones are integrated. Put each spec in the earliest batch its dependencies and collisions allow, with at most 5 specs per batch; split larger ones.

## 4. Ask how to run

Show the user the plan: the batches, with the number and title of each spec, and for each spec that can't run with another one, the reason in a few words (depends on `001`; changes the same file as `002`). Then ask which mode to use:

1. **One after the other**: one spec at a time, in numeric order, in the working directory.
2. **In parallel**: all at once when everything fits in one batch; otherwise batch by batch, as shown. Each spec in its own worktree.

The user can also rearrange the batches or pick only some specs; follow that, but refuse an arrangement that runs two colliding or dependent specs together, and say why.

Don't ask when there is nothing to choose: with a single spec, or when no two specs can run in parallel, say so and run them one after the other.

## 5. The cycle of a spec

Every spec goes through the same cycle, in either mode. What changes between the modes is where the work happens and how it is committed (steps 6 and 7).

### 5.1 Implement

Set the spec's `**Status:**` to `em andamento` and launch the implementation subagent. It knows nothing about this conversation; its brief must contain:

- The full text of the spec.
- The absolute paths of the ADRs to read before writing code, and the architecture summary from step 2.
- The stack and conventions from step 2, and the files this spec is expected to touch.
- What the specs it depends on already delivered (from their `## Implementação` sections), so it builds on that work.
- For a spec that was `em andamento`: that part may be done, and to check the code first.
- **Where to work**: the absolute path of the directory (the repository in sequential mode, the spec's worktree in parallel mode), and that every file it reads or writes and every command it runs must be inside it. In a worktree, also say that it holds only the tracked files: dependencies, build output and local files such as `.env` aren't there, and it may install dependencies inside the worktree if it needs them to build or type check.
- These rules:
  - Implement this spec only: its requirements and acceptance criteria, nothing from `Fora do escopo`, no extras.
  - Follow the ADRs and the existing architecture and conventions. If the spec can't be done without going against them, stop and report instead of improvising.
  - Don't write or change tests; a separate step does that. Don't delete or disable existing tests either.
  - Don't run git commands that change state: no `add`, `commit`, `stash`, `switch`, `checkout`, `reset`, `restore` or `push`. Leave the changes uncommitted.
  - Don't change anything inside `.raquinos-flow/specs/`.
  - Stay in the files of this spec. If another file has to change, make the smallest change possible and report it.
  - It may run the project's build, type check or linter to check its own work.
- The report to send back: files created and changed; what was done, in a few bullets; each acceptance criterion as met or not met, with the evidence; deviations from the spec and decisions taken; anything blocking.

### 5.2 Review

When the implementation subagent finishes, launch a **review subagent**, a different one, on Fable 5.2 at high effort, to look at the work with fresh eyes. It is read-only: say in its brief that it must not create, change or delete any file, nor run git commands that change state. Its brief must contain:

- The full text of the spec.
- The absolute paths of the ADRs, to read in full, and the architecture summary.
- The directory to review, and that the work is the uncommitted changes there: `git status --porcelain` and `git diff HEAD`, plus the content of the new files.
- The deviations and decisions the implementer reported, as claims to judge, not as facts.
- What to look for, in this order:
  1. **Architecture**: does the code respect the architecture of the project? Each piece in the right layer or module, dependencies pointing the right way, no business rule in the wrong place, no shortcut around a boundary, the same patterns the rest of the code uses.
  2. **ADRs**: does anything contradict a decision recorded in an ADR?
  3. **Spec**: is every requirement implemented and every acceptance criterion really met? Was anything done that the spec doesn't ask for, or that is listed in `Fora do escopo`? Were tests written or changed (they shouldn't be)?
- The report to send back: a verdict, `aprovada` or `ajustes necessários`; each acceptance criterion as met or not met, with the evidence; and the findings, each one with the file and line, what is wrong, which requirement, ADR or architecture rule it breaks, and whether it is **blocking** (must be fixed before the spec is concluded) or an **observation** (worth knowing, doesn't block). It reports only what it verified in the code.

### 5.3 Fix

If the review has blocking findings, launch a new implementation subagent (Opus 5.5, medium) to fix them, with the same brief as in 5.1 plus the findings and the instruction to fix those and nothing else. Then review again (5.2). Do at most two rounds of fixes.

### 5.4 Record

Update the spec file from the last review:

- Mark each acceptance criterion `[x]` only when the review confirms it is met; leave the others `[ ]`.
- Set `**Status:**` to `concluida` when the review is `aprovada` and every criterion is met. Otherwise leave it `em andamento`.
- Add (or replace) a section at the end of the spec, in Portuguese:

  ```markdown
  ## Implementação

  - O que foi feito, em poucos itens.

  **Arquivos:** `caminho/um`, `caminho/dois`

  **Commits:** `abc1234`, `def5678`

  **Revisão:** aprovada, com N rodadas de ajustes. Observações não bloqueantes, se houver.

  **Desvios e decisões:** o que saiu diferente da spec e por quê. Omitir se não houver.

  **Pendências:** o que falta para concluir. Omitir se não houver.
  ```

  Fill in `Commits` after the spec is committed (steps 6 and 7). `/flow:finish` copies this section into the PR/MR.

### 5.5 When a spec isn't concluded

A spec isn't concluded when its subagent reports a blocker, or when blocking findings remain after two rounds of fixes. Don't start any spec that depends on it; specs that don't depend on it go on. Show the user what is missing and ask:

1. **Try again**: one more round, with any guidance the user gives.
2. **Keep it as it is**: commit the partial work, leave the spec `em andamento` with its `Pendências`, and go on with what doesn't depend on it.
3. **Stop**: commit nothing more and leave everything where it is. Say where the uncommitted work is: the working directory, or the worktree's path.

In sequential mode, ask right away. In parallel mode, ask after every spec of the batch has gone through its cycle.

## 6. One after the other

For each spec, in numeric order: run the cycle (5.1 to 5.4) in the repository's working directory, then commit it by invoking the `flow:commit` skill with the Skill tool, passing the spec's number and title as the hint. It splits the work into commits by domain and pushes them. If it stops without committing (a failing hook, a push error), stop here too and report why.

Write the resulting commit hashes in the spec's `Commits` line, then go to the next spec. Each spec starts from a clean tree.

## 7. In parallel

For each batch, in order:

1. **Create the worktrees.** Note the commit the batch starts from (`git rev-parse HEAD`). For each spec of the batch, run `git worktree add -b flow-dev/spec-<NNN> .raquinos-flow/worktrees/spec-<NNN> HEAD`. `.raquinos-flow/` is ignored by git, so the worktrees don't show up in the project.
2. **Implement.** Launch the implementation subagents of the whole batch in a single message, each one working in its own worktree, and wait for all of them.
3. **Review.** Launch the review subagents of the whole batch in a single message, and wait for all of them. Run the fix rounds (5.3) the same way, in parallel for the specs that need them.
4. **Record** each spec (5.4), and resolve the ones that weren't concluded (5.5).
5. **Integrate**, one spec at a time, in numeric order. Skip a spec the user chose to stop on; its worktree stays.
   - Commit the work inside the worktree, with `git -C <worktree> add -A -- <paths>` and `git -C <worktree> commit`, following the same rules as `/flow:commit`: one commit per domain; message `type(domain): mensagem` in Portuguese, imperative, lowercase first letter, no trailing period, at most 72 characters; no `Co-Authored-By`, "Generated with" or any mention of Claude, Anthropic or AI; never `--no-verify`; leave out secrets and local artifacts (`.env*`, keys, installed dependencies, build output).
   - Bring the commits to the task branch: `git cherry-pick <batch start>..flow-dev/spec-<NNN>`.
   - **On a conflict**, don't resolve it: run `git cherry-pick --abort`, keep the worktree and its branch, set the spec back to `em andamento` with the conflicting files in `Pendências`, and tell the user. The specs that depend on it don't run.
   - After a clean cherry-pick, write the new commit hashes in the spec's `Commits` line, then remove the worktree and its branch: `git worktree remove --force .raquinos-flow/worktrees/spec-<NNN>` and `git branch -D flow-dev/spec-<NNN>`.
6. **Push** the task branch once the batch is integrated: `git push -u origin <branch>` when it has no upstream, otherwise `git push`. Skip it when the repository has no `origin`. If the push is rejected, don't force it: stop and show the error; the commits stay local.

The next batch creates its worktrees from the new `HEAD`, so it sees everything the previous batches delivered.

## 8. Report

In a few lines: each spec with its final status and commits; the review result and the non-blocking observations worth a look; deviations and decisions the user should review; specs left `pendente` or `em andamento`, and why; any worktree left in place, with its path. Suggest `/flow:finish` when every spec is `concluida`.
