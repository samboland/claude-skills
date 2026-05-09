---
name: goodnight
description: End-of-session closeout. Audits work done this session, reconciles forge issues vs TODO/DONE files vs actual code, closes what is truly complete, narrows residue, moves completed todos to done/, commits + pushes cleanup, leaves a status report. Anti-drift, anti-fake-completion. In boss-employee mode optionally posts a forge-comment digest so the boss can skim progress in the forge UI when not in Claude. Trigger:  "/team:goodnight", "sign off", "wrap up", "closeout", "going to bed", "end of night".
user_invocable: true
arg_description: '[--no-commit] [--no-digest] [--dry-run]'
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# /team:goodnight

Mind-set: proactive finisher. Reconcile reality (git + forge + files + code) and close everything that is genuinely done. Surface anything that looks half-done so the next session starts clean.

## Step 0: Read config + identify caller

Same as `/team:whatdoido` Steps 0-2.

## Step 1: Inputs

| Flag | Effect |
|---|---|
| `--no-commit` | Do the reconciliation but do not create commits. Useful when reviewing. |
| `--no-digest` | Skip the boss-comment digest in boss-employee mode (e.g. you're shutting down between sessions, or the boss has been running their own `/team:whatdoido` so the digest is redundant). |
| `--dry-run` | Print everything that would happen but make no writes, commits, or forge mutations. |

## Step 2: Capture this-session deltas

Define "this session" as: commits since the last push to origin, plus uncommitted work. Specifically:

- `git log origin/<default-branch>..HEAD --oneline` for committed-not-pushed.
- `git status --short` for uncommitted.
- `git diff origin/<default-branch>..HEAD --stat` for committed-not-pushed file deltas.
- `git diff --stat` for uncommitted file deltas.

Combine into a single "session work" list with file paths and one-line intent (parsed from commit messages where possible).

## Step 3: Audit todo/done state

For each file in `todo/`:
1. Does it appear "done by code" — was its scope shipped in a commit this session or recently?
   - Heuristic: look for the todo's NNNNN id in commit messages, branch names, or file paths. Read the file to compare its `Done when:` section against the diff if present.
2. Does its forge issue (if mirrored) still exist + still open?
3. Does its `**Status:**` line match observed state?

For each file in `done/`:
- Sanity check: does the corresponding forge issue (if any) show closed? If not, queue a forge-close action.

Build three buckets:
- **CLOSE**: file should move from `todo/` to `done/` AND its forge issue should close.
- **UPDATE STATUS**: file should stay in `todo/` but its `**Status:**` line is stale (e.g. "WIP" when the work is paused, or no status when it's actually `BLOCKED on X`).
- **LEAVE**: confirmed still in flight, no action needed.

## Step 4: Confirm with caller

Print a one-screen summary:

```
Session work: <N> commits, <M> files touched.
Audit results:
  - CLOSE: <K> todos
      todo/00012.md  Surface modifier badges  →  shipped in <sha>
      ...
  - UPDATE STATUS: <L> todos
      todo/00134.md  Circuit breaker         →  status "WIP" → "WIP <date>: kill_switches landed; waiting on Upstash rebuild"
      ...
  - LEAVE: <M> todos (truly in flight)

Proceed with all closures + updates? [y/N/edit]
```

If the user says `edit`, walk through each CLOSE and UPDATE STATUS with `AskUserQuestion` (one question per item, batched in 4 at a time). Defaults are the suggestions; user can demote, alter, or skip each.

If `--dry-run`: stop here. Print the plan, do nothing.

## Step 5: Execute

For each CLOSE:
- `git mv todo/NNNNN.md done/NNNNN.md`.
- Append a closing line to the file: `**Closed:** <date> — <one-line summary>`.
- Move the line in `TODO.md` to `DONE.md` under the current date heading. Create the heading if it doesn't exist.
- Close the forge issue: `gh issue close <N> --comment "Closed via /team:goodnight: <one-line summary>"` (or `glab issue close`).

For each UPDATE STATUS:
- Edit the `**Status:**` line on the file.
- Update the inline status tag on `TODO.md`.
- If the issue has a label change (e.g. `IP` → `Blocked (Internal)`), update via `gh issue edit --add-label / --remove-label` (or `glab` equivalent).

## Step 6: Commit

If `--no-commit`, skip. Otherwise:

```
chore(closeout): <N> closed, <M> status updates (<date>)
```

If push-on-commit is desired, push to origin. Default behavior: commit, do not push (let the user review in `git log` first; many users push manually after `/team:goodnight`).

## Step 7: Boss digest (boss-employee mode only, optional)

If `team.mode = "boss-employee"` and `--no-digest` is not set:

Build a digest of this session's work targeted at the boss. Useful when the boss is in the forge UI more than in Claude:

```
**Session digest — <date>**

Commits this session: <N> (<short-sha-list>)
Closed: <K> todos
  - #<N> <title> — <forge URL>
  - ...
In flight: <L> todos
  - #<N> <title> — <one-line current state>
Blocked: <M> todos
  - #<N> <title> — blocker: <reason>; needs: <what unblocks>
Questions for <boss>: <Q> open
  - <one-line Q text> — <questions-file>:Q<n>
```

Where to post:
- **Default**: as a comment on a long-lived "Session digest" forge issue. Skill creates this issue once if it doesn't exist (title: "Session digests" + label `team-digest`); subsequent runs append comments.
- **Alternative**: as a comment on whatever forge issue the boss most recently created. Detected via `gh issue list --author <boss-handle> --sort updated --limit 1`.

Choose default unless config has `[team.digest_target]` set.

The digest is markdown, posted via `gh issue comment <N> --body-file <tempfile>` (or `glab issue note create`).

## Step 8: Final report (to the caller)

Brief, in chat:

```
✓ Closed: <K> todos
✓ Status updated: <L> todos
✓ Committed: <commit-sha>
✓ Boss digest posted: <forge URL>   (boss-employee mode only)

Next session starts clean. <T> todos open: <C> critical, <H> high, <M> medium.
Top of next-session queue: <link to next move from a quick /team:whatdoido pull>
```

## Constraints

- **Anti-fake-completion.** Never close a todo just because it has a `WIP` tag and a recent commit. Read the `Done when:` section if present; require evidence the actual deliverable shipped. When in doubt, leave it open with a status update rather than close it.
- **Never close a todo whose owner is not the caller without explicit confirmation.** Especially in boss-employee mode: the employee should not close boss-owned scoping todos.
- **Never push without confirmation.** Commit yes (it's local), push no.
- **Never edit forge issues for the OTHER member without flagging it.** If an issue assigned to the boss has a stale status, surface it; don't mutate it.
- **Boss digest is optional.** `--no-digest` skips. Config `[team.digest_target] = "off"` disables it permanently.
- **Skill output uses natural English** even when caveman is on. Match `/team:whatdoido` voice.

## Related

- `/team:whatdoido`: read-only triage; what to look at next session.
- `/team:sync-tasks`: deeper sync; goodnight only handles this-session deltas.
- `/team:issue`: file new tasks the audit surfaced.
- `team-config.toml`: drives mode + digest behavior.
