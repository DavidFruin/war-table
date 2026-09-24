# Agent instructions

This is Dave's cross-project notes vault ("war-table"). It is separate from any project's code repo — clone it independently and reference it by path.

## When working on a project for Dave

1. Check `Areas/active-work.md` for another agent already on this project
   (see "Active work" below) *before* anything else — no point reading
   context for work someone else is mid-way through.
2. Check `Projects/<project-name>.md` (or `Projects/<project-name>/README.md` for projects that hold actual files, not just notes) for existing context: status, decisions, open questions, next steps.
3. If neither exists yet, create a note from `Templates/project.md`.
4. Claim your row in `Areas/active-work.md` and push it immediately (its own commit — see below), then do the work.
5. After meaningful progress (not every small edit), update the project's note: what changed, why, what's next. Keep it terse — bullet points, not prose logs.
6. Record decisions and their reasoning here, not just outcomes — future sessions (and Dave) need the "why."
7. Don't duplicate what's already derivable from the project's own git history or code. This vault is for things that *aren't* obvious from the code: intent, context, tradeoffs, external constraints.
8. Release your row in `Areas/active-work.md` once you stop, whatever the reason.

## Active work — avoiding collisions

Multiple agents, on different machines, can be working for Dave at the same
time. `Areas/active-work.md` is how they avoid duplicating or colliding with
each other — it's not a record, it's only ever "what's true right now."

- **Before starting substantial work on a project**, check
  `Areas/active-work.md` for an existing row on that same project. If one
  exists and looks current (see staleness below), don't just proceed as if
  it weren't there — either pick different, non-overlapping work, or surface
  the conflict to Dave rather than deciding for him.
- **Claim your row as soon as you start**, not after you finish: one row per
  active agent — machine/agent name, project, a short phrase for what,
  and a UTC timestamp. Commit and push this *immediately*, on its own, not
  batched with your eventual work commit — the whole point is other agents
  seeing it while it's still true.
- **Release your row when you stop** — done, blocked, or moving to something
  else. Remove it (or update it to the new thing), commit, push. Don't leave
  a stale claim sitting there.
- **Staleness:** a row more than 4 hours old is probably abandoned (a crash,
  a session that ended without cleanup), not a real lock. Treat it as a
  signal to check — a quick look at that project's recent git log settles
  whether it's genuinely still in progress — not as something that blocks
  you outright.

## Conventions

- One markdown file per project under `Projects/`, named after the project's repo or common name. Projects that hold actual files (configs, code snippets) rather than just notes get their own folder with a `README.md`.
- Use `[[wikilinks]]` to cross-reference other notes (Obsidian-style).
- This repo is public — never write credentials, tokens, private URLs, or anything sensitive here.
