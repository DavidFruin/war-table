# war-table

Dave's portable second brain: project notes, status, and decisions, as plain markdown.
Clone it anywhere, browse and link notes in [Obsidian](https://obsidian.md), edit raw files in Neovim, or point an AI coding agent at it.

## Layout

- `Projects/` — one file per project: status, decisions, links, next steps
- `Areas/` — ongoing responsibilities that aren't a single project
- `Inbox/` — quick captures to triage later
- `Templates/` — starting points for new notes
- `AGENTS.md` — how AI agents should read and update this vault

## Usage

```
git clone https://github.com/DavidFruin/war-table.git
```

Open the folder as a vault in Obsidian, or edit files directly with any editor.

**On a new machine:** also copy `Projects/claude-config/CLAUDE.md` to `~/.claude/CLAUDE.md` so Claude Code sessions there know about this vault too — it isn't automatic yet (see that project's notes).

This repo is **public** — don't put credentials, tokens, or anything sensitive in it.
