# Blackfire agent skills

Skills that teach coding agents how to use [Blackfire](https://blackfire.io) from the terminal.

> **Alpha.** These skills iterate fast and get refactored. Their content, file layout and names can change or break between commits, so treat nothing here as a stable interface: do not link to a path inside a skill or build tooling on its structure.

| Skill | What it covers |
|---|---|
| [`blackfire-cli`](skills/blackfire-cli/SKILL.md) | Profile an application and read profiles, SQL queries and call graphs with the `blackfire` CLI, and diagnose a run that produces no profile |

## Install

### With Skills.sh

```bash
npx skills add blackfireio/skills
```

### With Claude Code

```bash
/plugin marketplace add blackfireio/skills
/plugin install blackfire@blackfire
```

The plugin version stays at `1.0.0`. Updates track commits rather than releases, so reinstall or update to pick up the latest skill content instead of watching for a version bump.

This repository is a read-only mirror generated from the Blackfire monorepo: open an issue here for skill feedback, pull requests cannot be merged.

Product support: https://support.blackfire.upsun.com
