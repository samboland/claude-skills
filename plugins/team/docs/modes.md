# Modes

The `team` plugin supports two relationship modes. Pick one in `team-config.toml` `[team] mode = "..."`.

## peer

Two equal partners. Either member can:

- Assign a task to the other.
- File a question for the other.
- Decide scope on shared work.

**Configuration shape:**

```toml
[team]
mode = "peer"
members = ["Alice", "Bob"]

[forge]
default_assignee = ""   # empty = ask at creation
```

**Skill behavior in peer mode:**

- `/team:issue`: asks who the owner is at creation time. Default is the caller (caller files for themselves) unless they explicitly delegate.
- `/team:whatdoido`: shows the caller's queue + questions. "What's blocking the other" section appears when the caller owes the other an answer.
- `/team:goodnight`: produces a local DONE.md update + commit. No forge-comment digest by default (both members use Claude, so a forge comment is redundant).
- `/team:sync-tasks`: handles assignments to either member, no top-down assumption.

**Question docs:** single shared `QUESTIONS.md` file. Both members append questions, both answer. Sections by date, like a thread.

## boss-employee

Asymmetric. The boss sets scope, the employee executes. Subtle but real differences from peer mode:

- The boss may not use Claude at all — they live in the forge UI.
- Assignments default top-down (boss → employee).
- Status visibility for the boss has to flow through the forge, not local files.

**Configuration shape:**

```toml
[team]
mode = "boss-employee"
boss = "Rob"
employee = "Sam"

[forge]
default_assignee = "Sam"   # the employee, by default
```

**Skill behavior in boss-employee mode:**

- `/team:issue`:
  - When boss invokes: defaults owner to `employee` (assignment).
  - When employee invokes: defaults owner to `employee` (self-assignment); selecting `boss` as owner adds `kind:question-for-boss` label.
- `/team:whatdoido`:
  - Employee role: surfaces everything assigned-to-employee + everything unassigned (employee owns the default queue).
  - Boss role: surfaces decision/scoping tasks, plus any questions employee flagged for boss.
- `/team:goodnight`:
  - Employee invokes: same closeout as peer mode, **plus** posts a status digest as a forge-issue comment so the boss sees progress without opening Claude.
  - Boss invokes: closeout without digest (they're already in the forge).
- `/team:sync-tasks`:
  - Default `source_of_truth.mode = "forge-first"` recommended (so boss can edit issues directly and changes flow to the employee's local files).

**Question docs:** `QUESTIONS-FOR-<BOSS>.md`. One direction. Employee asks up; boss answers in the file or in the forge UI. The skill mirrors questions to the forge with `kind:question-for-boss` label so the boss sees them in their issue inbox.

**Boss-friendly affordances:**

- The session digest comment thread on a long-lived "Session digests" issue gives the boss a chronological feed of what shipped each session.
- Issues created via the GH UI directly (i.e. boss types in the browser) flow into the employee's local `todo/` on next `/team:sync-tasks` pull, with no Claude action required from the boss.
- Blocker chains use the `Blocked (External)` label so a `gh issue list --label "Blocked (External)"` query gives the boss a list of "things waiting on me to decide."

## Switching modes

Edit `team-config.toml` and run `/team:sync-tasks` to reconcile. The skills are config-driven at runtime; there is no migration step. But:

- Switching peer → boss-employee: re-decide `default_assignee`, decide whether to convert `QUESTIONS.md` to `QUESTIONS-FOR-<BOSS>.md` (rename + edit prefatory copy).
- Switching boss-employee → peer: invert `default_assignee` to empty, rename questions doc.

## When to pick which

- **Sole-founder with a contract dev**: boss-employee, founder = boss.
- **Two equal co-founders**: peer.
- **Junior dev working under a senior**: boss-employee, senior = boss.
- **Two friends building a side project together**: peer, even if one is more technical.
- **Father teaching son to code**: boss-employee — and that's the case this plugin was built for.

The plugin spirit: minimize friction for the person who isn't using Claude. In boss-employee mode that's the boss; in peer mode no one is privileged.
