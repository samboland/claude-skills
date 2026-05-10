---
name: hi
description: Read-only triage for the developer who invoked the skill. Identifies the caller via git config (with env var fallback), pulls their open issues from the forge + their open todos from local files, scans recent commits, and returns a ranked next-moves list. Config-driven; supports peer + boss-employee modes. Use when the developer asks "what should I work on" / "what do I do" / "/team:hi".
user_invocable: true
arg_description: '[--verbose]'
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# /team:hi

A "where am I, what's next" snapshot for whichever team member just invoked the skill. Read-only. No writes, no commits, no forge mutations.

## Purpose

Two-person dev teams have interleaved but distinct queues. After a context switch or a fresh session, it is easy to lose the thread of "which of my things is actually next." This skill cuts through it: pulls the caller's live state from git + the forge + the file ledger and hands back a short priority-ranked list.

## Inputs

| Flag | Effect |
|---|---|
| `--verbose` / `-v` | Include the raw issue list, commit digest, and full blocker chain. Default output is just the ranked moves. |

## Step 0: Read config

Read `team-config.toml` at the project root. If missing, stop and tell the user to copy it from `templates/team-config.toml.tmpl` in the team plugin.

Extract:
- `team.mode` (peer | boss-employee)
- `team.members` or `team.boss` + `team.employee`
- `team.identity_signals`, `team.env_var_name`, `team.identity_map`
- `forge.type` (github | gitlab) + `forge.repo`
- `source_of_truth.mode` (file-first | forge-first)
- `conventions.questions_file`
- `bmad.enabled` and (if present) `bmad.implementation_path`, `bmad.planning_path`, `bmad.sprint_status_file`. If `[bmad]` is absent or `enabled = false`, all BMAD-related steps below are no-ops.

Pick the forge CLI: `gh` for github, `glab` for gitlab. Verify it's installed and authed (`gh auth status` / `glab auth status`); if not, stop with a one-line install hint.

## Step 1: Sync with remote

Detect the branch state up front:
- **Default branch**: `git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's@^origin/@@'`. Fall back to `main` if the symbolic-ref isn't set (rare).
- **Current branch**: `git branch --show-current`. Empty means detached HEAD — print "detached HEAD; check out a branch and rerun" and stop.

Refresh:
- `git fetch --all --prune` (always; cheap and metadata-only).
- **If current branch == default branch**: `git status -sb` and read the ahead/behind line. If `behind N`, fast-forward with `git pull --ff-only`. If the working tree is dirty, `git stash -u`, pull, `git stash pop`. If fast-forward fails, stop and print a one-line "local + origin diverged, sync by hand."
- **If current branch != default branch** (i.e. you're on a feature branch): **do not pull.** Auto-pulling would either silently switch context or merge in unwanted commits. Just report the branch state — the caller knows whether they want to rebase / merge / pull manually.

Capture two anchors:
- **Default tip**: `git log -1 --format="%h %an %ar %s" origin/<default>` — what's the most recent commit on the release line.
- **Branch state**: `git rev-list --left-right --count origin/<default>...HEAD` for ahead/behind vs. default; `git rev-list --left-right --count origin/<current>...HEAD` for ahead/behind vs. own remote (if `origin/<current>` exists; if not, "branch not yet pushed").

Both anchors render in the opening lines so the caller sees their position relative to release AND their own published branch state.

A clean fetch is not proof of "no other-member activity." Negative claims about the other person's recent activity must cite a sha + relative time, not be asserted bare. See Step 7.

## Step 2: Identify the caller

In order:

1. **git_config**: read `git config user.name` and `git config user.email`. Lowercase + substring match against `team.members` (peer mode) or `team.boss` / `team.employee` (boss-employee mode). Apply `team.identity_map` overrides first if present.
2. **env_var**: if no match, read `$<env_var_name>` (default `$TEAM_USER`). Match against the same list.
3. **prompt**: only if both signals miss. Use AskUserQuestion with the team members as options.

In boss-employee mode, the caller is also tagged with their role (`boss` or `employee`) for downstream branching.

If the caller does not match any configured team member, stop and ask whether to add them to `team-config.toml`. Do not proceed with an unidentified caller.

Use the identity consistently for the rest of the run.

## Step 3: Pull open work from the forge

Run in parallel:

- **GitHub** (`forge.type = "github"`):
  - `gh issue list --assignee @me --state open --json number,title,labels,updatedAt --limit 50`
  - `gh issue list --label <questions-label> --state open --json number,title,labels,assignees,updatedAt --limit 50`, filter to entries assigned to the caller
- **GitLab** (`forge.type = "gitlab"`):
  - `glab issue list --assignee @me --opened --output json` (or equivalent flags; check `glab issue list --help` for the current API surface)
  - same filter for question-labeled issues

Aggregate into:
- `mine`: issues assigned to caller
- `questions_for_me`: questions awaiting caller's answer
- `my_blockers`: anything labeled `Blocked (Internal)` or `Blocked (External)` (label names from `[labels]` config)

## Step 4: Read the file ledger

Read:
- `TODO.md`: index. In peer mode, look for the caller's owner section if grouped by owner; otherwise scan all priorities. In boss-employee mode, scan all (employee inherits everything by default).
- `DONE.md` tail (last 20 lines): recent momentum / wins.
- `<questions_file>` from config: entries with `**Answer:** _(open)_` are things the caller is waiting on (in peer mode, the caller may also have unanswered questions FROM the other person — those are blocking the caller's response).

If `[source_of_truth].mode = "file-first"`, files are authoritative; forge mismatches are flagged in `--verbose` only. If `forge-first`, forge is authoritative; local file mismatches are flagged because they may indicate forge updates that haven't been pulled yet.

### BMAD planning layer (only if `bmad.enabled = true`)

Read `<bmad.implementation_path>/<bmad.sprint_status_file>` (typically `_bmad-output/implementation-artifacts/sprint-status.yaml`). Parse the `development_status` block.

Identify:
- **Active stories**: keys with status `in-progress` or `review`. These are story IDs (e.g. `7-1-franchise-sale-management`).
- **Next ready**: keys with status `ready-for-dev`.
- **Active epics**: keys matching `epic-N` with status `in-progress`.

For each active story, the corresponding file lives at `<bmad.implementation_path>/<story-id>.md`. Read the first ~30 lines to extract the story title and `Status:` line for cross-check.

This is read-only. The team plugin does not mutate sprint-status.yaml or story files — that's BMAD's responsibility, done via BMAD's SM/dev/qa workflows.

## Step 5: Read recent git activity

Two views, since the caller may be on a feature branch:

**Release-line activity** (always anchored at `origin/<default>`):
- `git log --author="<caller-email>" --since="7 days ago" --oneline origin/<default>`: what has the caller shipped to the release line?
- `git log --since="48 hours ago" --pretty=format:"%h %an %ar %s" origin/<default>`: what landed on the release line recently, with author + relative time.
- `git log --author="<other-member-email-or-login>" -n 5 --pretty=format:"%h %ar %s" origin/<default>`: the OTHER member's most recent release-line activity, used as the anchor for Step 7.

**Caller's branch progress** (only if current != default):
- `git log origin/<default>..HEAD --pretty=format:"%h %ar %s"`: commits on the feature branch not yet on the release line.
- `git status --short`: any uncommitted work? If yes, that's always the top of the next-moves list.

If on the default branch: skip the "branch progress" view; uncommitted work alone is enough.

## Step 6: Rank

Highest priority first; ties break on newest forge `updatedAt`:

1. **Uncommitted work in the tree.** Always top.
2. **Critical / P0** todos assigned to the caller.
3. **questions_for_me that unblock someone else's WIP.** Answering these is leverage.
4. **High / P1** todos NOT blocked on an external party.
5. **Blocked (Internal)** items where the corresponding question has been answered but the todo status hasn't been updated: low-cost cleanup.
6. **Medium / P2** todos.
7. **Blocked (External)**: at the bottom. Show them so the caller knows they exist; do not promote them above actionable work.

## Step 7: Output

**Voice override.** This skill's output is conversational, not compressed. Complete sentences, light humor where it fits, no hedging, no AI-slop ("moreover", "delve", "I'd be happy to", trailing "let me know if..."). Think friendly five-minute check-in, not a CLI dump.

**Every reference must be a working link.** Pair every todo / question / issue with its path or URL:
- Todos: `todo/00011.md`. Add the forge URL in parens: `todo/00011.md (https://github.com/owner/repo/issues/48)`.
- Questions: `<questions_file>` with section anchor.
- Strategy / docs: `docs/<file>.md` with section name after a hash.

Look up actual issue numbers from the Step 3 fetch; do not invent.

### Output shape

Open with one orienting line, then a branch-state block, then content. Branch state is two lines when the caller is on a feature branch, one line on the default branch.

```
Hey <Name> -- here's where you stand today. Last shipped to <default> <date> (<short-sha>), <N> commits in the past week.
origin/<default> HEAD: <sha> <author> <relative-time> <subject>
[only if on a feature branch:]
You're on `<branch>`: <N> commits ahead of origin/<default>, <M> ahead / <K> behind origin/<branch>  (or "branch not yet pushed").
[only if bmad.enabled and there's an active story:]
Active BMAD story: `<story-id>` -- <story-title> (<status>).  [If multiple: list each on its own line.]

## Uncommitted work
<either a clean-tree line or a bulleted file list with one-line intent>

## On your plate
- <N> todos assigned (<C> critical, <H> high, <M> medium)
- <Q> questions waiting on you
- <B> internal blockers / <X> external blockers
[only if bmad.enabled:]
- BMAD: <I> stories in-progress, <R> in review, <Y> ready-for-dev

## What's blocking <Other-Member>
(skip if nothing)
The other member cannot move on these until you answer or merge. Doing any is pure leverage.
- <Qn>: <question in natural sentence form> -- source: `<questions-file>` (forge URL). Blocks: `todo/NNNNN.md` if named.

## Ranked next moves
1. <Friendly sentence describing the move.>
   Why: <one-sentence reason>
   Links: `<file-path>` · <forge URL> · <related-todo>
2. ...

## Heads-up from the other side
<2-3 lines on what the other member shipped or moved in the last 24-48h.>

## Backlog (verbose only)
<full list with links>
```

If the tree is clean AND no todos AND no questions: write one acknowledging line and stop. Do not pad.

### Boss-employee mode tweaks

When caller role is `employee`:
- "On your plate" merges everything assigned to the employee with everything assigned to no one (employee owns the default queue).
- "What's blocking the boss" only appears if the employee owes the boss an answer or a merge. Most of the time this section is empty.
- "Heads-up from the other side" focuses on new issues the boss has filed since last session (boss-via-forge-UI is the primary inflow).

When caller role is `boss`:
- "On your plate" includes things the employee has flagged for the boss to decide (label `kind:question-for-boss`).
- "Ranked next moves" emphasizes scoping / decision tasks over execution.
- "Heads-up from the other side" focuses on what the employee shipped + any blockers they raised.

## Constraints

- **Read-only.** No writes, no `gh issue close`, no commits.
- **Never ask the user who they are unless both git_config and env_var miss.** Identity is on the machine.
- **Never recommend work owned by the other member.** If a task is assigned to the other person, it does not appear in the ranked list. It may appear in "What's blocking <Other>" only if the caller owes them something.
- **Never invent priority.** No Priority field → treat as Medium / P2.
- **Never promote external-blocked items above actionable ones.**
- **Negative claims about other-member activity must cite sha + relative time.** "Rob's last push was `dc128d3` two hours ago, nothing since" is fine. "Rob hasn't pushed in 48h" without a sha anchor is not.
- **Always render the `origin/<default>` tip anchor in the opening lines**, even on a clean fetch.
- **Never auto-pull when HEAD is on a feature branch.** Auto-pulling silently switches context or merges in unwanted commits. Fetch only; report state; let the caller decide whether to rebase / merge.
- **BMAD is read-only.** Surface active stories from `sprint-status.yaml`, but never edit it. Story state changes belong to the BMAD SM/dev/qa workflows, not this skill.
- **Skill output uses natural English** even when caveman mode is active session-wide. Deliberate carve-out.

## Related

- `/team:sync`: sibling skill for pushing/pulling between files and the forge.
- `/team:task`: builder for new tasks.
- `/team:bye`: end-of-session closeout.
- `/team:help`: read-only file-state snapshot + command list.
- `team-config.toml`: source of all configurable behavior.
