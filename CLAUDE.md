# raquinos-dev-kit

This repository is a Claude Code plugin marketplace. The catalog lives in
`.claude-plugin/marketplace.json` and each plugin lives in `plugins/<plugin>/`.

## Adding a plugin

1. Create `plugins/<plugin>/.claude-plugin/plugin.json` with `name`, `version`,
   `description`, `author`, `homepage`, `repository` and `license`.
2. Put each skill in `plugins/<plugin>/skills/<skill>/SKILL.md`, with `name`
   and `description` frontmatter.
3. Add a `README.md` at the plugin root.
4. Register the plugin in the `plugins` array of `marketplace.json`, with
   `"source": "./plugins/<plugin>"`.
5. Update the plugins table in the root `README.md`.

## Rules

- Plugin names are kebab-case and permanent once published. To change the
  label users see, set `displayName`.
- The entry `name` in `marketplace.json` must match the `name` in `plugin.json`.
- Names must not start with `claude-`/`anthropic-` or contain `claude` as a
  word (reserved, or triggers a warning).
- `source` paths start at the repository root and never use `..`.
- Bump `version` in `plugin.json` on every release, or users won't receive the
  update.
- Don't put a `CLAUDE.md` inside a plugin; instructions belong in skills.

## Validating

`claude` reads stdin, so redirect it to keep scripts from hanging:

```bash
claude plugin validate . </dev/null
claude plugin validate --strict ./plugins/<plugin> </dev/null
```
