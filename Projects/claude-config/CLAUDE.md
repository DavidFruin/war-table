# war-table notes

Dave keeps a cross-project notes vault at `~/war-table` (public repo: https://github.com/DavidFruin/war-table), synced via git and browsable in Obsidian. It holds project status, decisions, and context that isn't derivable from code or git history.

When working on one of Dave's projects:

1. Check `~/war-table/Projects/<project-name>.md` for existing context on the current project (status, decisions, open questions, next steps) before starting substantial work. If the vault isn't cloned on this machine, clone it: `git clone https://github.com/DavidFruin/war-table.git ~/war-table`.
2. If no note exists for the project yet, create one from `~/war-table/Templates/project.md`.
3. After meaningful progress (not every small edit), update the project's note: what changed, why, what's next. Keep it terse — bullets, not prose logs.
4. Record decisions and their reasoning, not just outcomes — future sessions need the "why," not just the "what."
5. Don't duplicate what's already obvious from the project's own code or git history.
6. This vault is a **public repo** — never write credentials, tokens, or anything sensitive into it.
7. After editing files in `~/war-table`, commit and push so the update is visible from other machines: `cd ~/war-table && git add -A && git commit -m "..." && git push`.
