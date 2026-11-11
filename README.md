# raquinos-dev-kit

A Claude Code plugin marketplace with Renato Aquino's development skills.

## Installation

Add the marketplace once:

```bash
claude plugin marketplace add renato-aq/raquinos-dev-kit
```

Then install the plugin you want:

```bash
claude plugin install <plugin>@raquinos-dev-kit
```

Or, from inside a Claude Code session:

```
/plugin marketplace add renato-aq/raquinos-dev-kit
/plugin install <plugin>@raquinos-dev-kit
```

## Plugins

| Plugin | Description |
| :- | :- |
| [flow](plugins/flow) | Task workflow for git: start a branch, commit with Conventional Commits, and finish with a PR/MR on GitHub or GitLab |

### flow

| Skill | What it does |
| :- | :- |
| `/flow:start [type] <description>` | Updates the default branch and creates `<type>/<short-description>` |
| `/flow:commit [hint]` | Splits the changes into one commit per domain as `type(domain): mensagem` (in Portuguese) and pushes after each commit. On the default branch, only `chore`/`docs`/`ci`/`build`/`style` go straight in |
| `/flow:finish [keep]` | Bumps the version (semver) if it wasn't bumped, opens the PR/MR to the default branch, waits for the pipeline, merges it and deletes the branch. `keep` keeps the branch. Conflicts are reported, never resolved |

## Layout

```text
.
├── .claude-plugin/
│   └── marketplace.json      # catalog: lists the plugins and where to fetch them
└── plugins/
    └── <plugin>/
        ├── .claude-plugin/
        │   └── plugin.json   # plugin manifest
        ├── README.md
        └── skills/
            └── <skill>/
                └── SKILL.md
```

## Development

Validate the marketplace and each plugin before pushing:

```bash
claude plugin validate .
claude plugin validate --strict ./plugins/<plugin>
```

To test locally the way a user would:

```bash
claude plugin marketplace add ./
claude plugin install <plugin>@raquinos-dev-kit
```

Edits to local plugins take effect in the next session or after `/reload-plugins`.

## License

[MIT](LICENSE)
