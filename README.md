# jj — Claude Code Skill for Jujutsu VCS

A [Claude Code](https://claude.ai/claude-code) skill that provides expert guidance for [Jujutsu (jj)](https://jj-vcs.dev/), a Git-compatible version control system with first-class conflict handling, automatic snapshots, and a working-copy-as-commit model.

## What's Included

- **SKILL.md** — Core reference: mental model, everyday commands, bookmarks, rebasing, conflict handling, simultaneous multi-branch editing, Git interop, common pitfalls
- **references/revsets.md** — Complete revset query language reference
- **references/filesets.md** — Fileset expressions for selecting files in diffs, splits, and squashes
- **references/templates.md** — Template language for customizing `jj log` output

## Installation

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/fjkorf/jj-skill.git ~/.claude/skills/jj
```

Then invoke with `/jj` in any Claude Code conversation.

## Topics Covered

- Working-copy-as-commit mental model
- Change IDs vs commit IDs
- Revsets, filesets, and templates
- Bookmarks (jj's branches)
- Squash, split, absorb workflows
- Simultaneous multi-branch editing via merge commits
- Rebasing all feature branches at once
- Conflict resolution (conflicts stored in commits)
- Operation log and undo
- Divergent commit recovery
- Git command equivalents

## License

MIT
