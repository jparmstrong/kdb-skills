# kdb-q Agent Skill

Agent skill for working with [kdb+/q](https://code.kx.com) — the time-series database and query language. Covers q syntax, qSQL, tables, joins, IPC, iterators, HDB architecture, and coding style.

## Install

Requires [Node.js](https://nodejs.org) for the `npx` command.

**Add to your current project:**
```sh
npx skills add jparmstrong/kdb-skills
```

**Install globally (available in all projects):**
```sh
npx skills add -g jparmstrong/kdb-skills
```

**Target a specific agent:**
```sh
npx skills add -a claude-code jparmstrong/kdb-skills
```

The skill is under `skills/kdb-q/` in this repo — the CLI discovers it automatically. Installs into your project's agent skills directory (e.g. `.claude/skills/kdb-q/`). Supported agents include Claude Code, Cursor, Copilot, OpenCode, Amp, and [40+ others](https://skills.sh).

To install only this skill from a repo containing multiple skills:
```sh
npx skills add -s kdb-q jparmstrong/kdb-skills
```

## Manage

```sh
npx skills list                        # see installed skills
npx skills update                      # update all skills
npx skills remove kdb-q                # uninstall
npx skills check                       # check for updates
```

## What's included

| File | Contents |
|------|----------|
| `SKILL.md` | Core q language rules, qSQL, tables, joins, IPC, gotchas, coding style |
| `references/datatypes.md` | Full type table, casting, nulls, attributes |
| `references/qsql.md` | select/exec/update/delete, functional qSQL, time-bucketing |
| `references/joins.md` | lj, ij, aj, wj and all variants with examples |
| `references/ipc.md` | Handles, sync/async, .z callbacks, patterns |
| `references/hdb.md` | Splayed/partitioned tables, .Q.dpft, EOD workflow |
| `references/iterators.md` | each, over, scan, each-prior, peach |

Reference files are loaded on demand — the agent reads them when the task requires it.
