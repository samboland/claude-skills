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

Run `/team:whatdoido`. Expect a "clean slate" message:

```
Hey <Your Name> -- here's where you stand today. Last shipped <date> (<sha>), 0 commits in the past week.
origin/main HEAD: <sha> ...

## Uncommitted work
Tree clean.

## On your plate
0 todos assigned, 0 questions, 0 blockers. Nothing to surface.
```

If you see "Unknown caller" or "team-config.toml not found", the config or identity setup is off — fix and rerun.

## Step 6: Bootstrap from existing forge issues (optional)

If the project already has GitHub/GitLab issues you want to mirror locally:

```
/team:sync-tasks --direction pull
```

This creates `todo/NNNNN.md` files for each open issue and `done/NNNNN.md` for each closed issue. Filenames are sequential starting at `00001`; the forge issue number is preserved in the file body via the `**Forge:** <url>` line.

After this, your local files mirror the forge state. From here on, the `source_of_truth.mode` setting determines which side is canonical.

## Step 7: First real task

```
/team:issue
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

**"Forge issue created but `**Forge:**` line not in file"**: rare race condition. Re-run `/team:sync-tasks --direction pull` to backfill.

**Boss-employee mode: "boss isn't seeing the digest"**: check `[team.digest_target]` in config and verify the long-lived digest issue exists in the forge (label `team-digest`). If missing, the next `/team:goodnight` recreates it.

## Two-person setup notes

Both members install the plugin (it's a plugin install per Claude Code instance). The config file in the repo is shared — both members read the same `team-config.toml`. Identity differs because `git config user.name` differs per machine.

If only one member uses Claude (boss-employee mode where the boss stays in the forge UI), only the employee installs the plugin. The boss never runs `/team:*` skills; they interact via the forge web UI and read the digest comments the employee's `/team:goodnight` posts.
