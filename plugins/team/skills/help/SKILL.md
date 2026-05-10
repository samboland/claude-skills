---
name: help
description: Read-only file-state snapshot + command list. Reads team-config.toml, identifies the caller via git config (no prompting), counts entries in TODO.md / DONE.md / todo/ / done/ / questions file, and prints the 5 /team:* commands with one-line behavior. Does NOT touch git history, the forge, or any network. Use when the developer asks "/team:help", "what does this plugin do", "how do I use the team skill", or wants a fast sanity check that config is being read correctly without paying for a forge round-trip.
user_invocable: true
arg_description: ''
allowed-tools:
  - Read
  - Glob
  - Bash
---

# /team:help

Lightweight, file-only orientation. Confirms config + identity, counts the local ledger, lists the commands. Designed to be cheap: no forge calls, no `git log`, no fetch.

## Purpose

Two situations this skill exists for:

1. **First-time setup verification.** After copying templates and editing `team-config.toml`, before paying for a `/team:hi` (which hits the forge), check that the config parses, the caller is identified, and the file structure is in place.
2. **Self-documenting commands.** When the developer forgets which `/team:*` command does what, surface the menu without doing real work. Cheaper than `/team:hi`, more accurate than the developer's memory.

## Step 0: Read config

Read `team-config.toml` at the project root.

If missing: print one line — "No `team-config.toml` at project root. Copy from the plugin's `templates/team-config.toml.tmpl` and edit." Stop.

If parse fails: print the parse error verbatim and stop. Do not attempt to guess defaults.

Extract:
- `team.mode` (peer | boss-employee)
- `team.members` OR `team.boss` + `team.employee`
- `team.identity_map`
- `forge.type` (github | gitlab) + `forge.repo`
- `source_of_truth.mode`
- `conventions.questions_file`
- `bmad.enabled` and (if present) `bmad.implementation_path`, `bmad.sprint_status_file`. If `[bmad]` is absent or `enabled = false`, skip BMAD counting in Step 2.

## Step 1: Identify the caller (best-effort, no prompting)

In order:

1. **git_config**: read `git config user.name`. Lowercase + substring match against `team.members` (peer) or `team.boss` / `team.employee` (boss-employee). Apply `team.identity_map` first.
2. **env_var**: if no git_config match, read `$TEAM_USER` (or whatever `env_var_name` says). Match the same way.
3. **unknown**: if both miss, render `caller = unknown` in the output. Do NOT prompt — `/team:help` is meant to be silent.

In boss-employee mode, also tag the caller with `(boss)` or `(employee)`.

## Step 2: Count file state (no parsing of bodies)

- `TODO.md` exists? If yes, count lines matching `^- \[ \]` (open todo lines). If no, render `MISSING`.
- `DONE.md` exists? If yes, count entries (lines starting with `- [`). If no, render `MISSING`.
- `todo/` exists? If yes, count `*.md` files (excluding `.gitkeep`).
- `done/` exists? If yes, count `*.md` files (excluding `.gitkeep`).
- `<questions_file>` exists? If yes, count lines containing `**Answer:** _(open)_`. If no, render `not yet created`.

If `bmad.enabled = true`:
- Read `<bmad.implementation_path>/<bmad.sprint_status_file>`. Parse the `development_status` block. Count epics by status (`backlog`, `in-progress`, `done`) and stories by status (`backlog`, `ready-for-dev`, `in-progress`, `review`, `done`).
- If the sprint-status file is missing despite `bmad.enabled = true`, render `BMAD: enabled in config but sprint-status.yaml not found at <path>`.

These are file existence + line counts only. No body parsing, no priority breakdown, no recent-activity scan. That's `/team:hi`'s job.

## Step 3: Print

One screen. Plain English, no emoji unless the developer's CLAUDE.md asks for them.

```
You: <name> (<role>) | unknown
Mode: <peer|boss-employee>
Forge: <github|gitlab> · <owner/repo>
Source of truth: <file-first|forge-first>

Files:
  TODO.md       <N open>  | MISSING
  DONE.md       <K done>  | MISSING
  todo/         <T files>
  done/         <D files>
  <questions>   <Q open>  | not yet created

[only if bmad.enabled:]
BMAD:
  Epics:    <E_total> total  (<E_done> done, <E_ip> in-progress, <E_bl> backlog)
  Stories:  <S_total> total  (<S_done> done, <S_review> in review, <S_ip> in-progress, <S_ready> ready-for-dev, <S_bl> backlog)
  Stories not yet done: <list of story IDs, max 5; "..." if more>

Commands:
  /team:hi    — Live triage. Identifies you, pulls forge issues, scans recent
                git activity, returns ranked next-moves. Use as your daily
                "what should I work on" entry point.
  /team:task  — File a new task. Walks title / owner / priority / status / body,
                writes todo/NNNNN.md, mirrors as a forge issue. The owner
                default depends on mode + role.
  /team:sync  — Reconcile local files ↔ forge. Direction defaults to
                source-of-truth (forge-first → pull, file-first → push).
                Run after editing issues in the forge UI to refresh local files.
  /team:bye   — End-of-session closeout. Audits commits vs. todos, closes what
                shipped, narrows residue, optionally posts a digest comment for
                the boss in boss-employee mode.
  /team:help  — This snapshot. File-only, no network.
```

If `caller = unknown`: append a one-line nudge:

```
Heads-up: your git user.name does not match any team member. Either fix git config
(`git config user.name "<name>"`) or add a `[team.identity_map]` entry mapping
your git name to a configured member name. /team:hi will refuse to run until this
is resolved.
```

If any file shows `MISSING`: append a one-line nudge:

```
Setup incomplete: <list of missing files>. Copy from the plugin's templates/.
```

## Constraints

- **Read-only, network-free.** No `git fetch`, no `gh`/`glab`, no commits.
- **Never prompt.** If identity can't be determined from git_config or env, say so and move on — do not ask the user.
- **Never invent counts.** If `TODO.md` is missing, the count is `MISSING`, not `0`.
- **Output is the same in caveman mode.** This is a reference card; voice is neutral and deterministic.
- **Fast.** No file should be opened more than once. No body parsing. No regex over directories larger than necessary.

## Related

- `/team:hi`: the live triage equivalent — same identity logic, but adds forge + git scans and a ranked queue.
- `/team:task`: file new tasks.
- `/team:sync`: reconcile files ↔ forge.
- `/team:bye`: end-of-session closeout.
- `team-config.toml`: drives every behavior.
