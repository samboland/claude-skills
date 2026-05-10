# team

Two-person dev team workflow as Claude Code skills. Drop into any project where two people share work; supports both peer-equal and boss-employee dynamics.

## What it gives you

- **File-based task ledger**: `TODO.md` index + `todo/NNNNN.md` detail files, `DONE.md` rollup. Permanent IDs, plain markdown, lives in the repo.
- **Forge sync**: bidirectional mirror with GitHub Issues (`gh`) or GitLab Issues (`glab`). Either side can be source of truth.
- **Identity-aware skills**: detect who's at the keyboard via git config; surface the right inbox.
- **Two relationship modes**:
  - **peer**: two equal partners; either can assign, either can ask the other a question.
  - **boss-employee**: assignment defaults top-down (boss → employee); a `/team:bye` digest can post to GH as a forge comment so the boss can track progress in the forge UI when they're not in Claude.

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
5. Run `/team:sync` to seed local files from existing GH issues (forge-first projects) or `/team:help` to confirm config is read correctly.

Full walkthrough: `docs/setup.md`.

## Skills

| Skill | Purpose |
|---|---|
| `/team:hi` | Read-only triage: identifies you via git config, pulls your open work from forge + files, returns a ranked next-moves list. |
| `/team:task` | Interactive builder for a new task: creates `todo/NNNNN.md` and a forge issue, sets owner per config + mode. |
| `/team:bye` | End-of-session closeout: reconciles forge ↔ files ↔ git, closes finished work, posts a status digest (boss-employee mode adds a forge comment so the boss sees it). |
| `/team:sync` | Manual sync between local task files and forge issues. Direction depends on `source_of_truth.mode`. |
| `/team:help` | Read-only file-state snapshot + command list. No git, no forge. Use for "where am I" without the network round-trips of `/team:hi`. |

## Modes

See `docs/modes.md` for the full matrix. Quick form:

|  | peer | boss-employee |
|---|---|---|
| Default issue assignee | unspecified, asked at creation | the employee |
| Status digest target | local DONE.md only | local DONE.md + (optional) forge issue comment so boss can skim in the forge UI |
| Question docs | one shared `QUESTIONS.md`, both contribute | `QUESTIONS-FOR-<BOSS>.md`, employee asks up |
| Source of truth | usually file-first | either; forge-first works well if the boss prefers editing in the GH UI directly |

## BMAD method integration (optional)

If your project uses [BMAD method](https://github.com/BMad-Code/BMAD-METHOD) for planning (PRDs, epics, stories, sprint plans), opt in by adding a `[bmad]` section to `team-config.toml`:

```toml
[bmad]
enabled = true
implementation_path = "_bmad-output/implementation-artifacts"
planning_path       = "_bmad-output/planning-artifacts"
sprint_status_file  = "sprint-status.yaml"
```

When enabled, the team skills become BMAD-aware **without managing BMAD state**:

- `/team:hi` surfaces the active BMAD story (status `in-progress` or `review`) in its opening lines, plus a "BMAD" line in "On your plate" with story-state counts.
- `/team:bye` notes when this session's commits touched BMAD story files or `sprint-status.yaml`. The boss-digest mentions BMAD progress alongside team todos. **Never edits BMAD files** — story-state transitions belong to the BMAD SM/dev/qa workflows.
- `/team:sync` excludes `<implementation_path>` and `<planning_path>` so BMAD story files never get mirrored as forge issues.
- `/team:help` shows epic + story counts by status.

### Task-as-pointer pattern

A team task doesn't have to *be* the work — it can be an **operational handle** pointing at planned BMAD work, with a goal stated in the task body. Add a `**BMAD reference:**` line to the body:

```markdown
# #00042 Implement BMAD story 7-1

**Owner:** sam | **Priority:** High | **Status:** WIP
**BMAD reference:** story `7-1-franchise-sale-management`
**Goal:** Get franchisee sale CRUD demoable end-to-end.

Plan:
- Run bmad-bmm-dev-story for 7-1
- Verify acceptance criteria
- bmad-bmm-code-review when ready
```

When a task carries a BMAD reference, `/team:hi` renders the referenced story's status next to the task in the ranked-moves list. `/team:bye` flags pointer tasks where the referenced story moved status this session, prompting "review whether to close." Closing the task is still a human decision — the task may have followup (docs, demo, cleanup) that outlives the story.

The two systems serve different layers: BMAD owns planned work (epics → stories with acceptance criteria), the team plugin owns operational work (ad-hoc tasks, bugs, decisions, questions, daily triage, *and* pointer tasks that orchestrate BMAD execution). They coexist; they don't merge.

## License

MIT.
