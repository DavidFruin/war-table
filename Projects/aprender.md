---
status: active
repo: https://github.com/DavidFruin/aprender
---

# aprender

## Summary
Dave's daily Spanish grammar and vocabulary practice: an owner-only page at
`/aprender/` on the [[simple-social]] site, linked from Settings' info block.
Written practice only, interface entirely in Spanish, Mexican + Spain Spanish
(vosotros included). Long-term goal: the whole grammar and the RAE's
vocabulary; near-term: comfortably define 20,000 words in 1-5 years.
Default session about 15 minutes.

## How it works
- PHP backend + plain JS/CSS page, all in the `aprender` repo. Its own SQLite
  file `private/aprender.db`, next to `userdata.db`.
- Login is Simple Social's: the page sends the site's own token, and
  aprender's `api.php` checks it with [[ssapi]]'s code; owner role only.
- The app only grades. Lessons are written ahead of time as JSON batches;
  once a week Dave downloads his database and Claude (skill
  `crear-lecciones`, in the repo) writes the next batch from the progress
  report. Dave uploads it on the page.
- Spaced repetition: FSRS-5. Missing accents count as wrong (setting).
  Every answer, lesson attempt, review and writing is kept.

## Status (2026-11-16)
- Built and tested locally: 84 unit checks, 25 HTTP checks on a local copy
  of ssapi, browser run-through at desktop and phone width. First batch
  `0001-diagnostico` (placement test) written; it simulates 114/114 correct.
- Commits: aprender up to d3b967c; ssreact 9f2d69a (Settings link);
  ssapi 08c57a4 (root.htaccess rule for /aprender/).
- **Not deployed.** `ssh el1` times out from Citadel (10.111.1.2:10102).
  To deploy to dev, from a machine that reaches el1:
  1. ssapi `deploy/root.htaccess` → `public_html/.htaccess` on dev (by hand).
  2. `aprender/scripts/deploy.sh dev`
  3. ssreact `scripts/deploy-web.sh dev` (Settings link).
  4. Open dev's `/aprender/`, Ajustes → Subir lote → `content/lotes/0001-diagnostico.json`.

## Open
- Frequency list for vocabulary (everyday vs formal text): undecided;
  batch 0001 uses hand-picked words without ranks.
- Default daily numbers (1 lesson, 10 new words) to revisit after the
  placement batch.
