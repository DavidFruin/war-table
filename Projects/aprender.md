---
status: active
repo: https://github.com/DavidFruin/aprender
---

# aprender

## Summary
Daily Spanish grammar and vocabulary practice, on its own site
https://aprender.davidfruin.com since 2026-11-19 (before that an owner-only
page at `/aprender/` on the [[simple-social]] dev site).
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
- Ranked list now exists (see 2026-11-18 below); still open: a placement
  test to find where Dave's gaps start, and making weekly batches draw from
  the list (plus a set share of Spain words). Waiting for Dave's go-ahead.
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
- 2026-11-18: map page (#mapa): every lesson on a dotted line with best grade (unanswered = wrong); current lesson can be started or skipped any time; done lessons can be retaken. New lessons must have exactly 10 questions per region (check.php). Answered fields locked.
- 2026-11-18: dictation has a 🇲🇽/🇪🇸 accent switch beside ♀/♂. Mexican voices Piper `es_MX-ald-medium` (man) and `es_MX-claude-high` (woman, by pitch; Dave to confirm by ear). word_audio has an accent column (schema v6, dev backup `private/aprender.db.pre-v6-20261118`); setting `acento_dictado` default mx. Batch 0001 re-recorded in both accents; `bin/grabaciones.php` adds recordings to an already-imported batch.
- 2026-11-18: Dave wants Mexico first, plus Spain's vocabulary and vosotros, and **nothing paid**. Built `content/vocabulario/` with `tools/vocabulario/construir.py` from free data only (Leipzig spa-mx web corpus, OpenSubtitles counts, Wiktionary via kaikki; ~1.4 GB in `~/aprender-datos`, not in the repo). Weights from Dave's ChatGPT spec (35/25/15/10/10/5) in `pesos.json`; "modern/practical" come only from Wiktionary labels. 30,000-lemma main list, 594-word Spain list, 530 Mexicanisms. Known weakness: no free Mexican *spoken* data, so slang ranks low (chido ~11k, órale ~29k).

## 2026-11-19: own site, aprender.davidfruin.com
- Dave asked for its own subdomain with "the exact same login" as Simple Social and all Simple Social users copied over. He chose: users from **app** (live), **copied once** (not shared live), and **everyone gets their own progress** with the same lessons; only the owner uploads batches.
- Virtualmin sub-server of davidfruin.com (dir dns web ssl logrotate), PHP **fcgid** like dev (it was created as FPM; switched), wildcard cert linked. Virtualmin put the vhost on the deprecated IPv6 `...b55::25` while DNS AAAA is `...f03c:95ff:fe9a:83b8`, so IPv6 got the wrong cert; fixed with `modify-domain --shared-ip6`. Watch for this on any new sub-server.
- Layout: `aprender/` (app), `ssapi/` (accounts only), `public_html/` (page + `api.php`, `auth.php` stubs + own .htaccess with an enforcing CSP), `private/` (.env with its own JWT_SECRET and SITE_NAME=Aprender; `userdata.db` accounts; `aprender/contenido.db` shared lessons/words; `aprender/usuarios/<id>.db` per learner). Deploy: `scripts/deploy.sh` with RSYNC_RSH for the private route.
- ssapi d0822b8: `SSAPI_ACTIONS` lets another site serve only some actions; `SITE_NAME`/`MAIL_FROM` brand the emailed codes. Defaults unchanged, so app/dev are unaffected (not redeployed there).
- 19 accounts copied from app with `scripts/copiar-usuarios.php` (read-only snapshot of app, ssapi's own migrations to v11, everything but accounts emptied; app's userdata.db untouched). Dave's dev aprender data split with `bin/separar.php` into contenido.db + usuarios/1.db; snapshot kept as `private/aprender-de-dev-20261119.db`.
- Verified live: page, 403 on private files, Spanish errors, social actions refused, a real token through Apache (throwaway account, deleted after). Not verified: an emailed code actually arriving (needs a real inbox), Dave's own login.
- Dave logged in and confirmed it works (2026-11-19). Then: dev's aprender files and databases moved to `/home/davidfruin/backups/aprender-dev-20261119/` (not deleted); dev's root.htaccess now 301s `/aprender*` to the new site (ssapi commit); the ssreact Settings link (still owner-only) points to the new site, deployed to dev only; davidfruin.com's links page (repo my-links) has "Aprender (learn Spanish)". That deploy also published Dave's uncommitted Kick-label fix in my-links, still uncommitted there.

## 2026-11-20: leaderboard, options, first-time steps
- Usernames (required before anything else, `nombres.db`) and a leaderboard page (Tabla): who did today's lessons, and streaks.
- Formality (`sintaxis.tratamiento`) is batch 0002's estructura topic at Dave's request; placement lesson `diag-tratamiento` (rewrite formally or informally) is live.
- Dave chose: the course is **Mexico / Latin America**; **España** (vosotros, leísmo, Spain words, Spain words in the vocabulary test) and **vos** are switches, off by default. Spain and vos content goes in lessons of their own (never mixed into general lessons). Accents count only in tilde lessons and punctuation only in exercises about it, unless the learner turns those on. Placement lesson `diag-voseo` (batch 0001-voseo) is live. Schema v9.
- New accounts must pick a username, look at the main settings, take the vocabulary test, then do every placement lesson before the app opens. Accounts with any answers before 2026-11-20 skip all of it. A placement lesson added later just goes first in line.
- Punctuation without AI: exact for lesson answers (optional commas written `[,]`); free writing could only get mechanical hints (¿ ¡ pairs, capitals), not built.
- 2026-11-20: "rewrite in the past" is a checked rule: every verb lesson has 1-2 transformar exercises worded "Pasa la frase al <past tense>: «…»". Vos questions now name the vos countries (Argentina, Uruguay, Paraguay, Central America).
- 2026-11-20 (schema 10): every wrong answer shows the rule it broke. Accent/spelling rules are found automatically (src/diagnostico.php); grammar rules come from the exercise's explicacion plus "fallos" (likely wrong answers with their rule, required on completar/conjugar from batch 0002). The weekly worksheet asks for a "regla" when a mistake got no specific rule, and the next batch must add it as a fallo correction. Word form sentences need an explicacion.
- 2026-11-20: "use it" word review shipped: each word gets 1-2 "usar" sentences (rewrite a sentence using the word), one of the turns in word reviews.
- 2026-11-20: anti-cheating priority (Dave): on touch screens answers are typed only on the page's own keyboard (system keyboard suppressed with inputmode=none; trusted input and paste blocked), because phone autocorrect/suggestions give answers away. Hardware keyboards on tablets are blocked too.
- Still pending Dave's go-ahead: a gender (género) placement lesson was offered.
- 2026-11-20: weekly report and guide judge each lesson by its **first try** (retakes don't count in scores or weak points; a lesson failed first and passed later is still weak). Dave's request.
- 2026-11-21: about **half of each weekly batch is review** (weak points first, then passed points with the lowest first-try score / taught longest ago), mixed through the week. Dave's request.
- 2026-11-20: sign-ups capped at **50 accounts** (`MAX_USERS=50` in the site's private/.env; ssapi f776a2c, unset elsewhere so app/dev unaffected; .env backup kept beside it). 21 accounts at the time. Database download is now owner-only, like batch upload. Deployed.
- Placement (0001) finished 2026-11-20; every topic but past tenses failed on the first try.
- 2026-11-21: **batch 0002 live** (`0002-irregulares-tratamiento-genero`): 15 lessons, 7 review + 8 new (present irregulars, tú/usted, two género topics: two because the 20 % género minimum can't be met with half the batch on review), 140 words from rank 6001 (redone the same day after Dave finished all four vocabulary tests + Verbos: ~17,500 of 30,000 words, ~2,960 of 4,673 verbs; the 8001–8140 words are saved in `content/vocabulario/reserva-8001-8140.json`). `verbo.indefinido.irregular` postponed to 0003. Vosotros can't be reviewed while Dave has España off. Other learners (Liz, Yanin, Jos, renatitapop, Sarah_Evans, annagomez) are still in placement, 0–3 of 8 done.
- 2026-11-20: **five vocabulary tests** (1-4 general, "verbos"), no word shared between them. General tests are scored together (each one's newest answers); verbs scored alone against the list's ~4,670 verbs. Hoy offers one a week; the test page lists all so any can be taken now (Dave wanted to do all at once). Schema 11 (vocab_tests.prueba); server backup `private/aprender.pre-v11-20261120/`. Deployed.
