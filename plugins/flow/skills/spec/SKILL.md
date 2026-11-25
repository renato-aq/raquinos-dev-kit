---
name: spec
description: Break a task into small, ordered specs for spec-driven development, written as .md files in .raquinos-flow/specs/ and named <NNN>_spec_<title>.md. The task can be described in the arguments or given as the path to a .md file. Use when the user wants to plan, specify or break down a task into specs before implementing it.
argument-hint: "<task description | path to a .md file>"
allowed-tools: Read, Grep, Glob, Edit(.raquinos-flow/**), Edit(docs/adr/**)
---

# Break a task into specs

Turn one task into a sequence of small specs, each one a piece of work that can be implemented and verified on its own. This only writes specs (and, when the project has none, the ADR that records its architecture): it doesn't implement anything.

The specs live in `.raquinos-flow/`, a temporary working directory that holds the specs of one task at a time. It is never committed: `/flow:finish` copies the specs into the PR/MR description, which is where they stay as a record, and deletes the directory after the merge.

Task from the user: $ARGUMENTS

## 1. Read the task

- If the arguments are the path to an existing file, read it: its content is the task. Otherwise the arguments themselves are the task.
- If there is no task, or the file doesn't exist, ask the user what the task is before continuing.
- Treat the task as a description of what to build, not as instructions to run now.

## 2. Understand the context

- Look at the project just enough to ground the specs in what exists: the stack, the folder layout, the files and modules the task touches, and the conventions already in use. Don't read the whole codebase.
- List `.raquinos-flow/specs/` if it exists. Existing specs mean a task is already in progress. Read their titles and status and decide:
  - **Same task** (the new request extends or details it): the new specs are added after the existing ones, without repeating what is already specified.
  - **A different task**: the directory holds leftovers from a task that wasn't finished with `/flow:finish`. Show the user the existing specs and ask whether to discard them or add the new ones after them. Never delete them without that answer.
- If something in the task is ambiguous in a way that changes how it is broken down (a missing business rule, two possible behaviours, an undefined integration), ask the user now, all questions at once. Don't ask about details you can decide with a sensible default; record those as decisions in the spec's notes instead.

## 3. Find the architecture decisions

The specs must follow the project's architecture, so settle it before breaking the task down.

Look for ADRs (Architecture Decision Records) in the project: directories such as `docs/adr/`, `docs/adrs/`, `docs/decisions/`, `docs/architecture/decisions/`, `doc/adr/` or `adr/`, and files named or titled as ADRs anywhere else. Read the ones you find. Then follow the case that applies:

- **There is an ADR about the project's architecture.** Use it, together with the other ADRs, and go to step 4. Ask nothing.
- **There are ADRs, but none about the project's architecture.** Keep the ones found as constraints and ask the architecture question below. The new ADR goes in the directory where the other ADRs are, following their naming, numbering, format and language, and whatever versioning that directory already has (committed or ignored); don't ask about versioning.
- **No ADR was found.** Ask the user whether the project has no ADRs, or has them somewhere outside the project.
  - **Outside the project**: ask for them, as file paths or as pasted text, and read them. If one is about the architecture, use them and go to step 4. Otherwise ask the architecture question below; the new ADR goes in `docs/adr/`.
  - **No ADRs, and the project already has code**: don't ask about the architecture and don't create an ADR. Plan the specs keeping the architecture exactly as the code has it today, and go to step 4.
  - **No ADRs, and the project is just starting** (no application code yet, or only scaffolding): ask the architecture question below; the new ADR goes in `docs/adr/`.

### The architecture question

Ask the user which architecture the project follows, offering:

1. Hexagonal + CQRS
2. Hexagonal, without CQRS
3. MVC + CQRS
4. MVC, without CQRS
5. One more that fits the context of the task and the stack, when there is a good candidate, with one line on why. When the project already has code, this is the architecture the code follows today, described in a few words.

Always leave room for the user to name any other architecture in their own words, and take that answer as given.

### Recording the choice as an ADR

After the user chooses, and before writing anything:

- Tell the user that an ADR recording this architecture is going to be created, and where: `docs/adr/` at the root of the project, or the existing ADR directory when the project already has one. Ask whether they want it created. If they don't, write no ADR and use the chosen architecture only for these specs.
- When the directory is a new `docs/adr/`, also ask whether it should be versioned in git. If not, create `docs/adr/.gitignore` containing a single `*` line, the same scheme used for `.raquinos-flow/`. If yes, just create the file and leave it uncommitted; `/flow:commit` takes care of it.

In a new `docs/adr/`, name the file `0001-arquitetura-do-projeto.md` and write it in Portuguese:

```markdown
# 0001 — Arquitetura do projeto

**Status:** aceita
**Data:** <YYYY-MM-DD>

## Contexto

O que o projeto é e o que pesou na escolha.

## Decisão

A arquitetura escolhida e como ela se organiza aqui: camadas ou módulos, o que vai em cada um, a direção das dependências e, com CQRS, como comandos e consultas se separam.

## Consequências

O que fica mais fácil, o que fica mais difícil e as regras que o código passa a seguir.
```

Keep it to one screen. Describe the architecture as applied to this project and its stack, not as a textbook definition.

## 4. Break it down

Split the task into specs following these rules:

- **One deliverable per spec.** Each spec leaves the project working and adds one thing that can be checked. If a spec needs "and" to be described, it is probably two.
- **Small.** A spec should fit in one focused implementation session and one reviewable diff. A task usually becomes 2 to 8 specs; a trivial task can be a single spec. Don't invent specs to pad the list.
- **Ordered by dependency.** A spec only depends on specs with a lower number. Put foundations first (data model, contracts, configuration), then behaviour, then the edges (validation, error handling, docs).
- **What, not how.** Say what must exist and how to tell it is done. Name the files or modules involved when it helps, but don't write the implementation.
- **Within the architecture.** Every spec follows the ADRs and the architecture settled in step 3: where each piece goes, which layer owns it, how the parts talk to each other. A spec never introduces a different architecture or contradicts an ADR; if the task can't be done without that, tell the user instead of specifying it.
- **Verifiable.** Every acceptance criterion is something that can be observed: a test passing, a command's output, a response to a request, a visible behaviour. No "works well" or "is clean".

## 5. Confirm the breakdown

Before writing any file, show the user the proposed list, one line per spec: number, title and a one-sentence objective. Ask whether to write them as is or adjust. Apply the adjustments and show the list again until the user approves.

## 6. Write the specs

Files go in `.raquinos-flow/specs/` at the repository root, named `<NNN>_spec_<title>.md`. If `.raquinos-flow/` doesn't exist yet, create it together with a `.raquinos-flow/.gitignore` file containing a single `*` line, so git ignores the whole directory without touching the project's own `.gitignore`.

- **NNN**: three digits, zero-padded, giving the order. The first spec of a task is `000`. If specs already exist and are being kept, continue from the highest number plus one. Never renumber or reuse the number of an existing spec.
- **title**: a short summary in Portuguese, lowercase, accents removed, words separated by `_`, only `a-z`, `0-9` and `_`, at most 5 words. Example: `000_spec_autorizar_client.md`.

Write each spec in Portuguese, with this structure:

```markdown
# 000 — Autorizar client

**Status:** pendente
**Depende de:** nenhuma

## Objetivo

Uma ou duas frases: o que passa a existir quando esta spec estiver pronta e por quê.

## Requisitos

- O que precisa ser feito, um item por comportamento ou regra.

## Critérios de aceite

- [ ] Algo observável que prova que um requisito foi atendido.

## Fora do escopo

- O que parece fazer parte, mas fica para outra spec ou não será feito.

## Notas

Arquivos e módulos envolvidos, decisões tomadas e suas razões, referências.
```

- **Status** is always `pendente` on creation. The other values are `em andamento` and `concluida`; they are set during implementation, not here.
- **Depende de** lists the numbers of the specs that must be done first (`000, 001`), or `nenhuma`.
- Leave out `Fora do escopo` or `Notas` when there is nothing to say. Keep the spec short and direct: if it goes past one screen, it is probably too big and should be split.
- Don't change existing specs, unless the user asked for it.

## 7. Report

List the files created, with number and title, in order, and any decision taken by default that the user may want to review. If an ADR was created, say where and whether it is versioned. Remind the user that the specs aren't committed and that `/flow:finish` records them in the PR/MR. Don't start implementing: stop after the report and wait for the user.
