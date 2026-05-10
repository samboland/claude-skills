# Setup

Adopting the `team` plugin in a new or existing project. ~10 minutes.

## Prerequisites

- Claude Code installed.
- `gh` (GitHub CLI) or `glab` (GitLab CLI) installed and authed for whichever forge the project uses.
- A git repo (local or with a remote already pushed).

## Step 1: Install the plugin

In Claude Code:

```
/plugin marketplace add samboland/claude-skills
/plugin install team@sam-claude-skills
```

The `/team:*` skills become available.

## Step 2: Drop config + ledger files into the project

From the project root:

```bash
# Copy templates from the installed plugin
TEAM_PLUGIN=$(claude plugin path team)   # adjust to actual command if different

cp $TEAM_PLUGIN/templates/team-config.toml.tmpl ./team-config.toml
cp $TEAM_PLUGIN/templates/TODO.md.tmpl ./TODO.md
cp $TEAM_PLUGIN/templates/DONE.md.tmpl ./DONE.md
mkdir -p todo done
```

(If `claude plugin path` doesn't exist or the install path differs, copy from `<claude-skills repo>/plugins/team/templates/`.)

## Step 3: Edit `team-config.toml`

Replace placeholder values:

- `[team] mode`: `"peer"` or `"boss-employee"`.
- `[team] members` (peer) OR `[team] boss` + `[team] employee` (boss-employee).
- `[forge] type`: `"github"` or `"gitlab"`.
- `[forge] repo`: `"owner/repo"` for GitHub, `"group/project"` for GitLab.
- `[forge] default_assignee`: empty for peer, the employee's name for boss-employee.
- `[source_of_truth] mode`: usually `"file-first"` for peer, `"forge-first"` for boss-employee.
- `[team.identity_map]`: only needed if `git config user.name` doesn't match a configured team member name.

## Step 4: Add the team block to CLAUDE.md

Open the plugin's `templates/CLAUDE-team-block.md` and copy its content into your project's `CLAUDE.md` (create one if you don't have one). This tells Claude how the file system + skills work so non-`/team:*` interactions stay coherent.

## Step 5: Verify

Run `/team:help`. Expect a "clean slate" message confirming config is read and you're identified:

```
You: <Your Name> (<role>)
Mode: <peer|boss-employee>
Forge: <github|gitlab> · <owner/repo>
Source of truth: <file-first|forge-first>

Files: TODO.md (0 open), DONE.md (0 done), todo/ (0), done/ (0), questions (0 open).

Commands:
  /team:hi    — triage your queue (live: forge + git + files)
  /team:task  — file a new task (file + forge mirror)
  /team:sync  — reconcile files ↔ forge
  /team:bye   — end-of-session closeout
  /team:help  — this snapshot
```

If you see "Unknown caller" or "team-config.toml not found", the config or identity setup is off — fix and rerun.

## Step 6: Bootstrap from existing forge issues (optional)

If the project already has GitHub/GitLab issues you want to mirror locally:

```
/team:sync --direction pull
```

This creates `todo/NNNNN.md` files for each open issue and `done/NNNNN.md` for each closed issue. Filenames are sequential starting at `00001`; the forge issue number is preserved in the file body via the `**Forge:** <url>` line.

After this, your local files mirror the forge state. From here on, the `source_of_truth.mode` setting determines which side is canonical.

## Step 7: First real task

```
/team:task
```

Walk through the prompts. The skill creates `todo/00001.md` (or the next unused id), updates `TODO.md`, and creates a forge issue. Verify both exist.

## Step 8: Commit the setup

```bash
git add team-config.toml TODO.md DONE.md CLAUDE.md todo/ done/
git commit -m "chore(team): adopt /team plugin"
git push
```

## Troubleshooting

**"team-config.toml not found"**: copy from the plugin's `templates/` and edit.

**"Unknown caller; expected <X> or <Y>"**: your `git config user.name` doesn't match any configured member. Either fix git config or add a `[team.identity_map]` entry mapping your git name to a configured member name.

**`gh: command not found`**: install GitHub CLI from `cli.github.com`. Authenticate with `gh auth login`.

**`glab: command not found`**: install GitLab CLI from `gitlab.com/gitlab-org/cli`. Authenticate with `glab auth login`.

**"Forge issue created but `**Forge:**` line not in file"**: rare race condition. Re-run `/team:sync --direction pull` to backfill.

**Boss-employee mode: "boss isn't seeing the digest"**: check `[team.digest_target]` in config and verify the long-lived digest issue exists in the forge (label `team-digest`). If missing, the next `/team:bye` recreates it.

## Two-person setup notes

Both members install the plugin (one Claude Code install per machine, per project as needed). The config file in the repo is shared — both members read the same `team-config.toml`. Identity differs because `git config user.name` differs per machine.

In boss-employee mode where the boss is comfortable in the forge UI for filing tickets, the digest comment from `/team:bye` gives them a passive "what shipped today" feed without needing to run skills themselves. They can still run `/team:hi` or `/team:task` when convenient — it's a preference, not a constraint.
