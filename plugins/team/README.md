# team

Two-person dev team workflow as Claude Code skills. Drop into any project where two people share work; supports both peer-equal and boss-employee dynamics.

## What it gives you

- **File-based task ledger**: `TODO.md` index + `todo/NNNNN.md` detail files, `DONE.md` rollup. Permanent IDs, plain markdown, lives in the repo.
- **Forge sync**: bidirectional mirror with GitHub Issues (`gh`) or GitLab Issues (`glab`). Either side can be source of truth.
- **Identity-aware skills**: detect who's at the keyboard via git config; surface the right inbox.
- **Two relationship modes**:
  - **peer**: two equal partners; either can assign, either can ask the other a question.
  - **boss-employee**: assignment defaults top-down (boss → employee); a `/team:goodnight` digest can post to GH as a forge comment so the boss can track progress in the forge UI when they're not in Claude.

## Install

```
/plugin marketplace add samboland/claude-skills
/plugin install team@sam-claude-skills
```

## Setup in a target project

In the project root:

1. Create `team-config.toml` (template at `templates/team-config.toml.tmpl` in this plugin).
2. Create `TODO.md` and `DONE.md` (templates included).
3. Create `todo/` and `done/` directories.
4. Add the team block from `templates/CLAUDE-team-block.md` to your project's `CLAUDE.md`.
5. Run `/team:sync-tasks` to seed local files from existing GH issues (forge-first projects) or `/team:whatdoido` to confirm config is read correctly.

Full walkthrough: `docs/setup.md`.

## Skills

| Skill | Purpose |
|---|---|
| `/team:whatdoido` | Read-only triage: identifies you via git config, pulls your open work from forge + files, returns a ranked next-moves list. |
| `/team:issue` | Interactive builder for a new task: creates `todo/NNNNN.md` and a forge issue, sets owner per config + mode. |
| `/team:goodnight` | End-of-session closeout: reconciles forge ↔ files ↔ git, closes finished work, posts a status digest (boss-employee mode adds a forge comment so the boss sees it). |
| `/team:sync-tasks` | Manual sync between local task files and forge issues. Direction depends on `source_of_truth.mode`. |

## Modes

See `docs/modes.md` for the full matrix. Quick form:

|  | peer | boss-employee |
|---|---|---|
| Default issue assignee | unspecified, asked at creation | the employee |
| Status digest target | local DONE.md only | local DONE.md + (optional) forge issue comment so boss can skim in the forge UI |
| Question docs | one shared `QUESTIONS.md`, both contribute | `QUESTIONS-FOR-<BOSS>.md`, employee asks up |
| Source of truth | usually file-first | either; forge-first works well if the boss prefers editing in the GH UI directly |

## License

MIT.
