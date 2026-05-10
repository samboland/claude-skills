---
name: task
description: Interactive builder for a new task. Walks the user through title, owner, priority, status, and body; validates against the team format rules; writes `todo/NNNNN.md`; updates `TODO.md`; mirrors to the forge as a new issue. Config-driven; supports peer + boss-employee modes. Use when the developer says "new todo", "open an issue", "/team:task", or wants to file a task.
user_invocable: true
arg_description: '[title]'
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# /team:task

Interactive builder for a new `todo/NNNNN.md` file plus its forge-issue mirror.

## Step 0: Read config

Same as `/team:hi` Step 0. Bail with a hint if `team-config.toml` is missing.

## Step 1: Identify the caller

Same as `/team:hi` Step 2. The caller is used as the default reporter, and (in peer mode) as a candidate default owner.

## Step 2: Gather inputs

If a title was passed as the argument, use it. Otherwise ask.

Ask via `AskUserQuestion` (or accept inline if the user offered values up front):

1. **Title** (free-form text). Must be self-contained: the title alone answers "what is this?". Reject titles like "fix the bug" or "update X" without subject. Suggest a rewrite if too vague.
2. **Owner**:
   - Peer mode: list `team.members` as options. Default = caller. The caller can pick the other member if delegating.
   - Boss-employee mode:
     - If caller is `boss`: default owner = `employee` (top-down). Boss can self-assign by overriding.
     - If caller is `employee`: default owner = `employee`. Employee can flag for the boss by selecting boss; this also adds the `kind:question-for-boss` label and the title gets prefixed `[FOR BOSS]` if it's a decision request rather than executable work.
3. **Priority**: Critical / High / Medium / Low / Nominal.
4. **Status** (default `none`): `WIP`, `BLOCKED: <reason>`, or none.
5. **Body**: free-form. Validate per `[conventions]` section below.

## Step 3: Validate

Before writing:

- **Em-dash check**: if `conventions.em_dash_ban = true`, reject U+2014 (em dash) and U+2013 (en dash) in the body. Suggest `:` for definition / appositive, `;` for clause-join, `.` for sentence break. Block until clean.
- **Title format**: `<title>` becomes `# #NNNNN <title>` in the file (single space between `#NNNNN` and title text). No em dashes in titles either if banned.
- **Title self-containment**: titles like "fix the bug" or "update the thing" without a subject get rejected with a rewrite suggestion. **Exception**: titles that point at BMAD work (e.g. "Implement BMAD story 7-1", "Run the qa workflow on epic-3") are self-contained even if the verb is generic, because the BMAD ID disambiguates. Don't reject these.
- **Owner is a real team member**: if owner doesn't match `team-config.toml`, bail.
- **Body length**: warn if under 30 chars (probably not enough context) or over 4000 chars (consider splitting). For pointer-pattern tasks (BMAD reference + Goal lines), 30 chars is too low — require at least the goal line.

## Step 4: Compute next ID

`ls todo/ done/ 2>/dev/null | grep -oE '^[0-9]{5}' | sort -n | tail -1`. Add 1. Zero-pad to 5 digits. If neither directory has any files, start at `00001`.

Reserve the ID by writing the file before the forge call.

## Step 5: Write the file

Path: `todo/<NNNNN>.md`. Format:

```markdown
# #<NNNNN> <Title>

**Owner:** <Owner> | **Priority:** <Priority> | **Status:** <status-or-`none`>

[optional, when the task points at planned work elsewhere:]
**BMAD reference:** story `<story-id>`  |  workflow `bmad-bmm-<name>`  |  epic `<epic-id>`
**Goal:** <one-sentence outcome the task is meant to achieve>

<Body>

## Notes

(none yet)
```

Then update `TODO.md`: insert a new `- [ ] [<Title>](todo/<NNNNN>.md) <status-tag>` line under the appropriate priority section.

### Task-as-pointer pattern

Tasks don't have to *be* the work. They can be operational handles that point at BMAD work (a story, a workflow, an epic) and state the goal of executing it. Examples:

- "Implement BMAD story 7-1 — get franchisee sale CRUD demoable" with `**BMAD reference:** story 7-1-franchise-sale-management`.
- "Run bmad-bmm-create-architecture for the messaging refactor" with `**BMAD reference:** workflow bmad-bmm-create-architecture`.
- "Close out epic-3 retrospective" with `**BMAD reference:** epic epic-3`.

The pointer pattern is recognized when the body contains a line starting with `**BMAD reference:**`. `/team:hi` will surface the referenced story's current status next to the task; `/team:bye` will flag when a referenced story moves status during a session. The pointer is a hint, not a binding — closing the task is still a manual decision.

When the user files an obviously-pointer-shaped task ("implement story 7-1", "run the qa workflow", etc.) and `bmad.enabled = true`, suggest filling in the `**BMAD reference:**` and `**Goal:**` lines via `AskUserQuestion`. Don't insist; some tasks named after BMAD work are still meant to be ad-hoc.

## Step 6: Mirror to the forge

Build the issue:

- **Title**: per `conventions.issue_title_format` (default `#NNNNN <title>`).
- **Body**: copy of the file body. If `conventions.em_dash_ban` is true, also run a humanize pass (em/en dashes to `:`, curly quotes to straight) on the issue body before posting; warn the caller if changes were made.
- **Labels**: `<priority-label>` from `[labels]`, plus `TODO`, plus any status label.
- **Assignee**: the owner's GitHub/GitLab handle. The handle mapping should be in `team-config.toml` `[team.identity_map]` reversed; if not present, ask via `AskUserQuestion` and offer to write the mapping back to the config.

Run:

- **GitHub**: `gh issue create --title "<title>" --body-file <tempfile> --label <labels> --assignee <handle>`
- **GitLab**: `glab issue create --title "<title>" --description-file <tempfile> --label <labels> --assignee <handle>`

Capture the issue URL.

## Step 7: Cross-link

Append the forge URL to the file:

```markdown
**Forge:** <URL>
```

Commit the file + TODO.md update with a single small commit:

```
chore(todo): #NNNNN <Title>
```

(If the user is in the middle of other uncommitted work, ask before committing.)

## Step 8: Boss-employee specifics

When caller is `employee` and owner is set to `boss`:
- Add label `kind:question-for-boss` in addition to `TODO`.
- Append a one-line "for boss to decide" prefix to the body so the boss sees scope at a glance.
- The skill mentions in the final report: "Filed for <boss-name>; will surface in their forge inbox."

When caller is `boss` and owner is `employee`:
- Add label `assigned-to-employee` (or skip if redundant with the assignee field).
- The skill mentions: "Filed and assigned to <employee-name>; they will see it on their next `/team:hi`."

## Step 9: Final report

One short block:

```
Filed: todo/<NNNNN>.md
Forge: <URL>
Owner: <Owner>
Priority: <Priority>
Branch: <current-branch>  (only if commit was made and current != default)

Next: <one-sentence what-to-do hint based on owner + priority>
```

The forge issue exists immediately regardless of which branch the file commit landed on — the boss / other member sees the issue in GitHub the moment Step 6 finishes. If the file commit is on a feature branch, the local `todo/NNNNN.md` only reaches `<default>` when the branch merges; that's expected, since the forge issue is the canonical record in forge-first projects.

## Constraints

- **Reserve the ID before the forge call.** Otherwise an ID race or a forge-API failure can leave the file ID and forge-issue number out of sync.
- **Never auto-commit when the working tree has unrelated dirty changes.** Ask first, default no.
- **Em-dash ban applies to both file body and forge body** when the config is set.
- **Title must be self-contained.** Reject "fix it" / "update the thing" with a rewrite suggestion.

## Related

- `/team:sync`: pulls forge-side new issues that didn't go through this skill.
- `/team:hi`: surfaces the new task on the owner's next triage.
- `/team:bye`: end-of-session closeout.
- `/team:help`: read-only file-state snapshot + command list.
- `team-config.toml`: drives every behavior above.
