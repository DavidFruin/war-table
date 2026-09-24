---
name: war-table
description: Read and update Dave's cross-project notes vault (his "war table") at github.com/DavidFruin/war-table, cloned locally to ~/war-table. Use this whenever starting substantial work on one of Dave's own projects (his repos, his apps, his infrastructure) to load existing status, decisions, and open items before diving in — and again after making meaningful progress, to record what changed and why. Trigger this even if Dave doesn't mention the vault, war-table, or notes by name — any time you're about to do real work on a project of his, check first whether he has context recorded for it. Do not trigger for one-off questions, exploratory reads, or trivial edits with nothing worth recording.
---

# war-table

Dave runs Claude Code on several machines against the same projects. This
vault is how those sessions — and Dave himself — stay in sync on status,
decisions, and open items that aren't derivable from a project's own code or
git history. Another session may have written to it minutes ago; always pull
before reading, and push after writing, or your read is stale and your write
is invisible elsewhere.

The vault's own `AGENTS.md` is the source of truth for *what* to write and
*how* to structure it — read it once per session, the first time you touch
the vault, rather than assuming these instructions have kept up with it.
This file only handles getting you in and out correctly.

## Before starting substantial work on one of Dave's projects

1. **Locate the vault**, checking in this order:
   - `~/war-table` (the default location)
   - If neither exists, clone it: `git clone https://github.com/DavidFruin/war-table.git ~/war-table`
2. **Pull first**, always, even if you cloned it moments ago in a different
   step: `cd ~/war-table && git pull`. Another machine may have pushed since.
3. **Read `AGENTS.md`** at the vault's root if you haven't already this
   session — it defines the note structure, tone, and what counts as worth
   recording. Follow it exactly rather than improvising a different shape.
4. **Read the project's note**: `Projects/<project-name>.md`, or
   `Projects/<project-name>/README.md` for a project that holds actual files
   (configs, snippets) rather than just notes. Match `<project-name>` to the
   repo's name first; if that note doesn't exist, check for a looser name
   match before concluding there isn't one.
5. If no note exists at all, that's fine — proceed with the work. Create one
   (from `Templates/project.md`) once there's something worth recording, not
   preemptively.

## After meaningful progress

"Meaningful" means a decision was made, something shipped, a bug was found,
or the plan changed — not every file edit. Use judgment; when genuinely
unsure whether something rises to that bar, err toward recording it rather
than silently dropping context a future session would want.

1. Update the project's note per `AGENTS.md`'s conventions.
2. Pull again before committing — time has passed since step 2 above, and
   another machine may have pushed in the meantime. Resolve any conflict by
   keeping both sides' substance (this is append-heavy notes content, not
   code; a clean merge is almost always possible by hand).
3. Commit and push:
   `git add -A && git commit -m "<project>: <one line>" && git push`
4. If the push is rejected (someone else pushed first), pull, resolve, and
   push again rather than force-pushing.

## Hard rules

- **Never write credentials, tokens, API keys, internal URLs, or anything
  sensitive.** This repo is public. If a note would need a secret to be
  useful, describe where the secret lives instead of including it.
- Don't duplicate what a `git log` on the project's own repo already shows.
  This vault is for intent, context, and tradeoffs that aren't in the code.
- Don't narrate this skill's own mechanics (cloning, pulling, pushing) inside
  a project note — that's process, not project context.

## Setting this skill up on a new machine

The skill file itself lives in the vault, at
`Projects/claude-config/skills/war-table/`, so it updates the same way
everything else in the vault does — no separate distribution step needed.
One-time setup per machine:

```
git clone https://github.com/DavidFruin/war-table.git ~/war-table   # if not already cloned
mkdir -p ~/.claude/skills
ln -s ~/war-table/Projects/claude-config/skills/war-table ~/.claude/skills/war-table
```

That symlink is the whole mechanism. From then on, editing this file in the
vault (from any machine) changes what every machine's Claude Code sees —
there's nothing to manually re-copy.
