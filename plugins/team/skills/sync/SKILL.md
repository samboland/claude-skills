---
name: sync
description: Sync local task files (TODO.md, DONE.md, todo/*.md, done/*.md) with the project's forge issues (GitHub via `gh` or GitLab via `glab`). Direction depends on `source_of_truth.mode` in team-config.toml. Tier-1 implementation is one-shot pull (forge → files) for forge-first projects; one-shot push (files → forge) for file-first; full bidirectional reconciliation marked TODO. Trigger: `/team:sync`, "sync tasks", "pull issues", "push todos to github".
user_invocable: true
arg_description: '[--dry-run] [--direction pull|push|auto]'
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# /team:sync

Sync the local task ledger with the forge. One-shot, idempotent, opinionated about which side wins on conflict (per config).

## Step 0: Read config + identify caller

Same as `/team:hi` Steps 0-2.

## Step 1: Pick direction

`--direction` flag overrides; otherwise read `source_of_truth.mode`:

| `source_of_truth.mode` | Default direction | Meaning |
|---|---|---|
| `file-first` | `push` | Local files → forge issues. New/changed files create/update forge issues. Forge-only changes flagged (not pulled). |
| `forge-first` | `pull` | Forge issues → local files. New/changed forge issues create/update local files. Local-only changes flagged (not pushed). |

`--direction auto` does the opposite of the default (a manual reconciliation tool: e.g. forge-first project occasionally needs to push a batch of file edits up). Confirm before running.

## Step 2: Snapshot both sides

In parallel:

- **Files**: read `TODO.md`, `DONE.md`, every `todo/*.md`, every `done/*.md`. For each, extract: id, title, owner, priority, status, body, forge-url-if-known.
- **Forge**:
  - GitHub: `gh issue list --state all --json number,title,body,labels,assignees,state,updatedAt,url --limit 200`
  - GitLab: `glab issue list --all --output json` (with state filter)

Match files ↔ forge by:
1. `**Forge:** <url>` line in the file body (most authoritative).
2. ID-in-title regex: title like `#NNNNN ...` ↔ filename `NNNNN.md`.
3. Title text fuzzy match (Levenshtein) as last resort, only flagged if confidence > 0.85.

Build three lists:
- **paired**: file + forge match found.
- **file-only**: file exists, no matching forge issue.
- **forge-only**: forge issue exists, no matching file.

## Step 3: Pull direction (`forge-first` → file-first sync, OR `--direction pull`)

For each `forge-only`:
- Pick next available NNNNN id (max of `todo/`, `done/` + 1).
- Create `todo/<NNNNN>.md` from the issue body. Map labels → `**Owner:** / **Priority:** / **Status:**`.
- Append `**Forge:** <issue-url>` to the file.
- Add a line to `TODO.md` under the appropriate priority section.
- If the issue is closed, write to `done/<NNNNN>.md` instead and append a line to `DONE.md` under the close date (`updatedAt` truncated).

For each `paired`:
- If forge issue updated more recently than the file (`updatedAt` vs `git log -1 --format=%cI -- <file>`):
  - Refresh file body from issue body.
  - Refresh status / labels.
- Else: leave file alone.
- If issue state changed (open → closed), `git mv todo/NNNNN.md done/NNNNN.md` and update both index files.

For each `file-only`:
- Flag in the report: "File exists locally without a forge mirror. Run `--direction push` if you want it to live on the forge."
- Do not delete the file.

## Step 4: Push direction (`file-first` → forge sync, OR `--direction push`)

For each `file-only`:
- Create a new forge issue from the file body. Title format from `conventions.issue_title_format`. Labels from `[labels]` config + the file's priority/status. Assignee from `**Owner:**`, mapped to forge handle via `team.identity_map`.
- Append `**Forge:** <issue-url>` to the file.

For each `paired`:
- If file updated more recently (per `git log -1 --format=%cI -- <file>`) than issue (`updatedAt`):
  - Update issue body, title, labels, assignee from file state.
- Else: leave issue alone.

For each `forge-only`:
- Flag in the report: "Forge issue exists without a local file. Run `--direction pull` if you want it surfaced locally."
- Do not delete the issue.

## Step 5: Validate before writing (push direction only)

If `conventions.em_dash_ban` is true, run a humanize pass on the body before posting:
- U+2014 → `:`
- U+2013 → `:`
- Curly quotes → straight

Warn the caller if any conversions happened. Block only if the title contains an em dash (titles must be clean).

## Step 6: Report

Brief output:

```
Mode: <pull|push>
Source of truth: <file-first|forge-first>

Pulled: <N> new files from forge
Pushed: <M> new issues to forge
Updated paired: <P> entries (forge → file: <X>; file → forge: <Y>)
Closed by sync: <K> (issues moved to done/, todos removed from TODO.md)

Flags:
  - <Z> file-only entries (no forge mirror)
  - <W> forge-only entries (no local file)

Run /team:sync --direction <opposite> to handle the flags.
```

## Step 7: Commit (push direction only)

If files changed locally (new files written, refreshed bodies, moved to done/):

```
chore(sync): <N> issues mirrored to forge (<date>)
```

Pull direction: changes are file writes; commit similarly.

If `--dry-run`: skip writes + commits, just print the plan.

## Tier 2 (defer, not implemented in this version)

- **Conflict resolution UI**: when both sides updated since last sync and the diffs disagree, prompt for which to keep. Currently the side with the later timestamp wins.
- **Comment sync**: pulling forge issue comments back into a `comments/` subdirectory or appending to the file body.
- **Question-doc bidirectional sync**: peer-mode shared `QUESTIONS.md` ↔ forge labels.
- **Branch protection awareness**: skip files matching open PRs / merge requests so the sync doesn't trample WIP.

These are noted in the file body as `# TIER 2:` comments where future implementations should land.

## Constraints

- **Idempotent.** Running twice in a row should produce no second wave of changes.
- **Never delete a file.** If a forge issue was deleted, flag it ("issue #N existed last sync, gone now") rather than auto-delete the local file.
- **Never delete a forge issue.** Same direction.
- **Em-dash humanize is body-only on push.** Titles fail-fast.
- **Default `--direction` matches the source of truth.** The user opting into the opposite is explicit and confirmed.
- **Use `git log -1 --format=%cI -- <path>` for file mtime, not filesystem mtime.** The latter is unreliable across clones.

## Related

- `/team:task`: creates a single new task with its forge mirror in one shot. Use this for one-offs; `/team:sync` is the batch reconciliation.
- `/team:hi`: read-only triage on what's currently open.
- `/team:bye`: end-of-session closeout that includes a narrower sync.
- `/team:help`: read-only file-state snapshot + command list.
- `team-config.toml`: drives every behavior.
