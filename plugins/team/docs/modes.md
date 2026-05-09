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

- Assignments default top-down (boss → employee).
- The boss often prefers the forge UI for filing tickets even when they also use Claude. The plugin makes that comfortable: tickets the boss files in the GH UI flow into the employee's local `todo/` on the next sync; the employee never has to ask the boss to "use the right tool."
- Status digests can mirror to a forge comment so the boss tracks progress without opening Claude.

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
  - Employee invokes: same closeout as peer mode, plus optionally posts a status digest as a forge-issue comment so the boss can skim in the forge UI when they're not in Claude.
  - Boss invokes: closeout without digest (they can run `/team:whatdoido` themselves).
- `/team:sync-tasks`:
  - Default `source_of_truth.mode = "forge-first"` works well when the boss prefers filing tickets in the GH UI; either side wins, just pick one.

**Question docs:** `QUESTIONS-FOR-<BOSS>.md`. One direction. Employee asks up; boss answers in the file or in the forge UI. The skill mirrors questions to the forge with `kind:question-for-boss` label so the boss sees them in their issue inbox.

**Affordances for the boss:**

- The session digest comment thread on a long-lived "Session digests" issue gives a chronological feed of what shipped each session, readable in the forge UI without booting Claude.
- Issues filed in the GH UI flow into the employee's local `todo/` on next `/team:sync-tasks` pull; the boss isn't forced to use Claude to file work.
- Blocker chains use the `Blocked (External)` label so a `gh issue list --label "Blocked (External)"` query lists "things waiting on the boss to decide."

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
