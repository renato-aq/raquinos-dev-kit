# flow

A task workflow for git that works with both GitHub and GitLab.

| Skill | What it does |
| :- | :- |
| `/flow:start [type] <description>` | Updates the default branch and creates `<type>/<short-description>` |
| `/flow:commit [hint]` | Splits the changes into one commit per domain as `type(domain): mensagem` (in Portuguese) and pushes after each commit. On the default branch, only `chore`/`docs`/`ci`/`build`/`style` go straight in |

## Installation

```bash
claude plugin marketplace add renato-aq/raquinos-dev-kit
claude plugin install flow@raquinos-dev-kit
```
