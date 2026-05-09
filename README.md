# Claude Skills

Custom Claude Code skills by Sam Boland, distributed as a plugin marketplace.

## Installation

Add this repo as a marketplace and install the skills you want:

```
/plugin marketplace add samboland/claude-skills
/plugin install malware-static-analysis@sam-claude-skills
/plugin install team@sam-claude-skills
```

That's it. The skill becomes available as a slash command.

## Available Skills

### `/team:*`

Two-person dev team workflow plugin. Bundles `whatdoido`, `issue`, `goodnight`, and `sync-tasks` skills for managing tasks across local files and GitHub/GitLab issues. Config-driven (no hardcoded names); supports peer-equal and boss-employee modes.

See [`plugins/team/README.md`](plugins/team/README.md) for full documentation, or [`plugins/team/docs/setup.md`](plugins/team/docs/setup.md) for the 10-minute adoption walkthrough.

### `/malware-static-analysis`

Automated static malware analysis pipeline for Windows PE binaries (`.exe`, `.dll`). Runs pefile, YARA, and Ghidra headless decompilation, queries VirusTotal, and generates a comprehensive markdown report.

**STATIC ANALYSIS ONLY** — the target binary is never executed.

**Usage:**
```
/malware-static-analysis <path-to-binary> [--no-ghidra] [--vt-key:<key>]
```

**Requirements (on the analyst's machine):**
- Python 3 with `pefile` and `yara-python` (`pip install pefile yara-python`)
- Ghidra (optional, for decompilation) — set `GHIDRA_HOME` or install to a standard location
- VirusTotal API key (optional) — pass via `--vt-key:<key>` or put `VIRUSTOTAL_API_KEY=...` in a `.env` file in the working directory

**What it produces:**
- Full PE header analysis (sections, imports, exports, resources, signatures)
- String extraction with IOC categorization
- YARA rule matching
- Ghidra decompilation with targeted reading of interesting functions
- VirusTotal enrichment
- Markdown report with executive summary, behavioral hypothesis, MITRE ATT&CK mapping, and IOCs

**What it detects:**
- PEB-walking + hash-based API resolution (FNV-1a, DJB2, ROR13)
- DLL sideloading attacks
- Shellcode runners, packers, opaque predicate obfuscation
- Anti-debug/anti-VM techniques
- Multi-file package triage (distinguishes malware from padding/cover DLLs)

## Structure

This repo is a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces.md):

```
claude-skills/
├── .claude-plugin/
│   └── marketplace.json          # Lists all plugins
└── plugins/
    ├── malware-static-analysis/
    │   ├── .claude-plugin/plugin.json
    │   └── skills/malware-static-analysis/
    │       ├── SKILL.md
    │       └── ghidra_scripts/DecompileAllFunctions.java
    └── team/
        ├── .claude-plugin/plugin.json
        ├── README.md
        ├── skills/{whatdoido,issue,goodnight,sync-tasks}/SKILL.md
        ├── templates/{team-config.toml.tmpl,TODO.md.tmpl,DONE.md.tmpl,CLAUDE-team-block.md}
        └── docs/{modes.md,setup.md}
```

## License

MIT — see [LICENSE](LICENSE).
