# claude-worklog

A [Claude Code](https://claude.com/claude-code) skill that writes your daily timesheet entry for you.

It scans everything you did on a given day — commits, PRs, reviews and comments on GitHub, plus Jira tickets and Confluence pages when the Atlassian MCP is connected — asks you about the work that leaves no trace (meetings, pairing, research), and suggests how many hours to log, with the reasoning shown.

```
> /worklog yesterday
```

```
# Worklog — Tuesday, 2026-06-02

## Summary
Day focused on the gateway validation fix (PROJ-512): 3 commits, PR #142 opened and
ticket moved to review. Also reviewed PR #138 and joined sprint planning (~1h).

## Hours
- Active window: 09:20–17:40 (8h20), ~1h10 of breaks removed
- Planning (1h) inside the window, not added twice

Range ≈ 6.5–7.5h · Suggested: 7h

## Ready to paste
PROJ — Gateway validation fix (PROJ-512): 3 commits, PR #142 opened, ticket moved
to review; review of PR #138 (list pagination); sprint planning. [7h]
```

## Install

As a plugin:

```
/plugin marketplace add JibranMachiavelli/claude-worklog
/plugin install worklog@claude-worklog
```

Or copy the skill:

```bash
git clone https://github.com/JibranMachiavelli/claude-worklog
cp -r claude-worklog/skills/worklog ~/.claude/skills/
```

## Requirements

- [`gh`](https://cli.github.com) authenticated (`gh auth login`)
- Optional: Atlassian MCP connected in Claude Code for Jira and Confluence

## Why

Filling in hours at the end of the day means reconstructing it from memory. Your tools already know most of it. This skill collects it, fills the gaps with two questions, and gives you a text ready to paste.

## License

MIT
