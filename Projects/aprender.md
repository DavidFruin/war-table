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
- **Deployed to dev** (2026-11-16): root.htaccess (old copy kept as
  `.htaccess.bak-2026-11-16`), aprender, and the ssreact web build. Checked over
  HTTP: page, assets, redirect, 401 when logged out, src/ not served.
- **Batch 0001-diagnostico imported on dev (2026-11-17, Citadel)**: 8 points, 8 lessons,
  71 exercises, 20 words. Run with the page's own `validateBatch()`/`importBatch()` as
  the site user, since there is no CLI import. Backup first:
  `private/aprender.db.pre-0001-20261117`. Not on prod (aprender isn't there yet).
- From Citadel, `el1` (10.111.1.2) times out. The same server answers on
  10.111.2.1:10102 (same host key). Deploy with
  `RSYNC_RSH="ssh -o Hostname=10.111.2.1 -o HostKeyAlias=[173.255.247.77]:10102"`,
  then `chown -R davidfruin:davidfruin` on what was copied, since the copy runs as root.

- **Focus and weekly instructions (2026-11-17)**: verbs and sentence structure
  first, vocabulary second, accents/punctuation/writing last, as numbers in
  `content/guia/prioridades.md`, enforced by `bin/check.php`. The grading and
  lesson-writing rules are in `content/guia/`, written for any AI model; the
  `calificar` and `crear-lecciones` skills point there. Commits eff866d, 006090c.

## Open
- Frequency list for vocabulary (everyday vs formal text): undecided;
  batch 0001 uses hand-picked words without ranks. Until chosen, batches use
  estimated ranks, the least model-proof part of the weekly rules.
- Default daily numbers (1 lesson, 10 new words) to revisit after the
  placement batch.

## 2026-11-17 (later)
- "Seguir aprendiendo": after the daily goal the page offers to keep going (lessons, words and reviews past the daily limits). Deployed to dev.
- Gender of nouns is now a main focus: area `genero` (20-35 % of every batch), a 10-topic track in `content/guia/prioridades.md`, fill-in templates in `lecciones.md` (gap + hint `(el / la)` or `(frío)`), enforced by check.php. Batch 0002 will be the first with it.

- "Corrige" now says how many errors the sentence has (exercise field `errores`, DB schema v2; dev DB upgraded, backup `private/aprender.db.pre-v2-20261117`). Prod not yet.
- 2026-11-18: words now have `frases`, one fill-in sentence per form (verbs: 6-8, at least 4 tenses; nouns 3; adjectives 4); reviews show `frases_por_palabra` (default 4) at once, least recently seen first. One wrong of 3+ = "hard". DB schema v3 on dev (backup `private/aprender.db.pre-v3-20261118`). Batch 0001 words still have one sentence each.
- 2026-11-18 (later): Dave disliked the fill-in word review; replaced by **dictation** (phone Spanish voice via Web Speech API, setting `voz_dictado` default es; Spain/Mexico-only words use their own accent). The per-form `frases` feature was removed (table left empty in schema v3). Definir card unchanged.
- 2026-11-18 (correction): Dave wanted to keep the per-form sentences, only with the word visible. Now word reviews alternate: **forms** (lemma shown at top + `frases_por_palabra`=3 sentences, each a different tense/gender/number; first check after intro) and **dictation**. Batch rules for `frases` restored.
- 2026-11-18: added "¿Con qué letra?" word exercise (src/letras.php finds b/v, ll/y, c/s/z, r/rr, g/j, h, sc, x spots from the lemma; up to 4 blanks). Word reviews rotate forms → dictation → letters, skipping what a word lacks.
- 2026-11-18: letters exercise has no buttons: options shown as hints, the whole word is typed and graded with highlighted mistakes.
- 2026-11-18: dictation plays **recorded words** (Dave's browser has no Spanish voice). Piper `es_ES-sharvard-medium`, speakers M/F → setting `voz_dictado` = ambas/mujer/hombre. MP3 ~5 KB each in table `word_audio` (schema v4, dev backup `private/aprender.db.pre-v4-20261118`). PHP is jailed (can't exec), so `tools/audio.py` runs outside the jail as davidfruin (`runuser`); Piper venv + model in `~davidfruin/piper/`. Cron to run it every 5 min was **not installed** (blocked; awaiting Dave). Without a recording the page falls back to the browser voice.
- 2026-11-18 (later): Dave chose per-week recording over a server cron. `tools/audio.sh` (Piper in `~/piper` on the machine making the batch) writes each word's MP3s into the batch's last field `audio`; upload stores them in `word_audio`; check.php REGLA if a word lacks them; guide step added. Server Piper install and the cron idea removed; the 20 batch-0001 words keep their server-made recordings.
- 2026-11-18: tense names in forms (`forma`) and in prompts' parentheses are tappable buttons showing a short Spanish explanation (TIEMPOS_DEF in app.js; a test keeps it equal to TIEMPOS in reglas.php).
- 2026-11-18: dictation has a ♀/♂ switch (plays that voice, saves `voz_dictado`).
- 2026-11-18: a lesson failed 3 times (SALTAR_TRAS_FALLOS) can be skipped (button on the failed result and at the lesson's restart). Table `lesson_skips`, schema v5 (dev backup `private/aprender.db.pre-v5-20261118`). Skipped point still gets a grammar review card; informe.php lists skips and prioridades.md says to re-teach the point differently.
