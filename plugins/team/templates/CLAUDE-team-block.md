# Team workflow

Paste the section below into your project's `CLAUDE.md` (or include it via reference). It tells Claude how this project's task system works so the `/team:*` skills behave correctly.

---

## Task tracking (TODO.md / DONE.md)

This project uses the `team` skill plugin. Configuration lives in `team-config.toml` at the repo root.

- **`TODO.md`** at repo root: one-line index, grouped by priority. Each line links to a detail file in `todo/NNNNN.md`.
- **`DONE.md`** at repo root: one-line dated ledger of completed work. Each line links to `done/NNNNN.md`.
- **`todo/NNNNN.md`** and **`done/NNNNN.md`**: detail-per-task. Filename is a 5-digit sequential ID. IDs are permanent: when a task completes, `git mv todo/NNNNN.md done/NNNNN.md` and move its line from TODO.md to DONE.md under the current date. Don't re-use numbers.
- **Status tags inline in TODO.md** next to each line: `WIP`, `BLOCKED: <reason>`, `<date>`.
- **Owner metadata** on every todo/done file: `**Owner:** <name>` line near the top, drawn from `team-config.toml` members.

**Source of truth** is set in `team-config.toml` `[source_of_truth] mode`:
- `file-first`: TODO/DONE files canonical; forge issues mirror.
- `forge-first`: forge issues canonical; local files are a read-cache. New issues created in the forge UI surface to local files on next `/team:sync-tasks` pull.

**Skills** (loaded by `team` plugin):
- `/team:whatdoido`: read-only triage. Returns ranked next-moves for whoever is at the keyboard.
- `/team:issue`: interactive builder for new tasks. Creates the file + the forge issue.
- `/team:goodnight`: end-of-session closeout. Reconciles forge ↔ files ↔ git, closes finished work.
- `/team:sync-tasks`: manual sync between local files and the forge.

**Title format**: `# #NNNNN <Title>` (single space after `#NNNNN`). Titles must be self-contained so the title alone answers "what is this?": the reader should not need to open the issue body to understand scope.

**Mode-specific**:
- `peer` mode: questions live in a single shared file (default `QUESTIONS.md`). Either member can ask the other.
- `boss-employee` mode: questions live in `QUESTIONS-FOR-<BOSS>.md`. The employee asks up; the boss answers in the file or in the forge UI. `/team:goodnight` posts a forge-comment digest after each session so the boss sees progress without opening Claude.

If you see `kind:question-from-employee` or `kind:question-for-boss` labels, that's the team plugin's mirror of the questions file in the forge.
