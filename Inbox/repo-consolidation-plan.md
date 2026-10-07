---
status: proposal
written: 2026-10-03
for: Sonnet 5 (medium effort), implementing agent; GitHub settings steps for Dave
repos: ssreact @ 865ff6f (+ branch deploy-layout @ 466367f), simple-social-cli @ 73431e4, simple-social-cli-interactive, simple-social-tui, ssreact-native, sselectron, simple-social, sstests, ssapi
---

# Repo consolidation plan: four Simple Social repos

> **Progress as of 2026-10-07:**
> - **Step 1:** half done. The archive READMEs were pushed on 2026-10-03, but `ssreact-native` and `sselectron` are **not archived yet**; Dave needs to archive them.
> - **Step 2.0 done** (2026-10-06): `deploy-layout` merged into ssreact `master` as `466367f`.
> - **Step 2.6 done** (2026-10-06): ssreact deployed to dev's `public_html/app/`, replacing the vanilla frontend.
> - **Prod switched on 2026-10-06:** app.davidfruin.com runs ssapi + ssreact in the new layout, release **2.0.0 "Elia"** (tag `v2.0.0` in ssapi `236a338` and ssreact `466367f`). The vanilla frontend is retired everywhere. **So Step 5 (archive `simple-social`) can happen now**, and Step 3.6 no longer touches `download.html`.
> - **Since then:** ssreact `0379b5b` (version display + `/history` page, 2026-10-07) is on `master`; it's not recorded whether it's deployed to prod yet.
> - **Step 2 done 2026-10-07** (2.1–2.5 and 2.7): ssreact is a pnpm workspace with the app in `web/`, `scripts/deploy-web.sh` added and run once against dev. Dave chose to start without the prod real-use checks. Deviation from 2.2: the lockfile was **kept and migrated**, not deleted, because regenerating it upgraded dependencies (the main bundle came out 28% smaller, a different build). The restored lockfile reproduces the old build file for file.
> - **2026-10-07 (gh CLI):** archived `simple-social` (README replaced with an archive notice first), `ssreact-native` and `sselectron`, which finishes Steps 1 and 5. Created the empty public repo `DavidFruin/ssterminal` (Step 3.0). Step 3's import is not started.
> - **Step 3 done 2026-10-07:** `ssterminal` imported with history (CLI at the root, `wizard/`, `tui/`), submodules removed, one top-level Makefile (`make`, `make cli|wizard|tui`, `install-*`, `PREFIX`/`DESTDIR`, `make test`). The shared `lib/` was identical between the old pin `d91b04b` and `73431e4`, so 3.4 was a no-op. Verified on a local ssapi bench in the new layout: 35/35 CLI tests, 26/26 wizard tests, the TUI login screen, and install/uninstall into a temp prefix. Old repos got archive READMEs; All three were archived by gh on 2026-10-07. The ssreact Download page now points at ssterminal, deployed to **dev only** (prod waits for Dave's go).
> - **Steps 4 and 6 are not started.**

**Decided by Dave, 2026-10-03.** Simple Social ends up in **four repos**:

| Repo | Contents | Language | Visibility |
|---|---|---|---|
| **ssapi** | Backend (unchanged by this plan) | PHP | public |
| **ssreact** | Every React client: `web/` (today's ssreact), `mobile/` (React Native, Expo), `desktop/` (Electron), `packages/core/` (shared logic) | TypeScript | private |
| **ssterminal** | Every terminal client: `lib/` (shared C library + vendored headers), `cli/`, `wizard/`, `tui/`, one top-level `Makefile` | C | public |
| **sstests** | Test suites (unchanged, apart from updated references) | — | private |

Everything else is **merged or archived**:

| Repo | What happens | When |
|---|---|---|
| `ssreact-native` | Archived; empty except a README. Its future code is `ssreact/mobile/` | Now (Step 1) |
| `sselectron` | Archived; empty except a README. Its future code is `ssreact/desktop/` | Now (Step 1) |
| `simple-social-cli` | Merged into `ssterminal` (its history becomes ssterminal's root history), then archived | Step 3 |
| `simple-social-cli-interactive` | Merged into `ssterminal/wizard/` with history, then archived | Step 3 |
| `simple-social-tui` | Merged into `ssterminal/tui/` with history, then archived | Step 3 |
| `simple-social` | Archived. Prod and dev stopped using it on 2026-10-06 (both run ssapi + ssreact); it keeps the 1.x history (`v1.0.0`) | Now (Step 5) |

**Why (summary of the discussion with Dave):**
- The wizard and the TUI each embed the *whole* `simple-social-cli` repo as a git submodule, just to share `lib/`.
- The phone plan needed a copy-and-sync script to share ssreact's logic.
- The desktop plan needed a git submodule.

One repo per language removes all three workarounds: shared code becomes a normal folder, and a feature that touches several clients is one change. Industry-normal for a solo developer: one repo per product area / language, with folders, never long-lived branches.

---

## 0. Rules for the implementing agent
- One step per session where possible. Stop after each step and report.
- **Check `Areas/active-work.md` first.** Another agent ran overnight on ssapi/ssreact on 2026-10-02/03. Don't restructure ssreact while anyone else is working in it. Claim your row.
- **Keep git history** for everything that moves (details per step). Verify with `git log --follow` on a few moved files.
- **GitHub settings are Dave's:** creating `ssterminal`, archiving repos, renaming. An agent can commit README changes with push access, but `gh repo archive` / `gh repo create` need admin. Ask Dave to run them, or to confirm the agent may.
- **Never deploy anything to `app.davidfruin.com`.** Nothing in this plan needs a deploy except the optional install-docs update (Step 3.6), which Dave deploys.
- The war-table vault is public, so don't paste tokens or credentials anywhere.

---

## Step 1: archive the two empty repos (15 minutes, Dave or an agent + Dave)
1. In `ssreact-native` and `sselectron`, replace the README with:
   > **Archived 2026-10-03.** This app lives in the [ssreact](https://github.com/DavidFruin/ssreact) repo now (`mobile/` for the phone app, `desktop/` for the desktop app). See the war-table notes for the plans.

   Commit and push.
2. **Dave:** archive both: GitHub → Settings → Archive this repository, or `gh repo archive DavidFruin/ssreact-native` and `gh repo archive DavidFruin/sselectron`.

---

## Step 2: turn ssreact into a pnpm workspace (agent, about half a day to a day)
Do this **before** the phone port starts (its Phase 1 creates `packages/core` inside this structure).

**2.0 Done 2026-10-06.** `deploy-layout` was merged into `master` (`466367f`), so ssreact ships no `.htaccess`.

**Before 2.1 (required checks):**
- **Nobody else is working in ssreact.** On 2026-10-07 a row for the version/history work was still in `Areas/active-work.md`, even though that work had finished (war-table `e071175`, ssreact `0379b5b`). Ask Dave to confirm it can be removed; don't edit another agent's claim yourself.
- **Prod has been checked with real use since the 2026-10-06 switch:** a browser login, a post with a photo, a push notification from a second account, and opening from an old home-screen icon (`/app.html#/feed` should land on `/feed`). The record says these weren't done. If they still aren't, ask Dave before restructuring, so a prod problem isn't confused with fallout from the restructure.
- Start from the latest `master`, which includes `0379b5b`.

**2.1 Move the app into `web/`, keeping history:**
```bash
git checkout master && git pull
mkdir web
git mv src public index.html vite.config.ts tsconfig*.json eslint.config.js components.json package.json web/
# Leave at the root: .git*, .github/, README.md, pnpm-lock.yaml (regenerated below)
```
Check `git ls-files` for anything else that belongs to the app, such as `postcss` configs or `.env.example`, and move it too. `src/lib/versions.ts` (the release list behind the `/history` page) moves with `src/`; update the pointer to it in the [[simple-social]] note's Versions section to `web/src/lib/versions.ts`.

**2.2 Workspace files at the root:**
- `pnpm-workspace.yaml`:
  ```yaml
  packages:
    - web
    - mobile
    - desktop
    - packages/*
  ```
  `mobile`, `desktop` and `packages/*` don't exist yet; pnpm ignores missing globs.
- Root `package.json`:
  ```json
  {
    "name": "ssreact-workspace",
    "private": true,
    "scripts": {
      "dev": "pnpm --filter @ss/web dev",
      "build": "pnpm --filter @ss/web build",
      "lint": "pnpm -r lint"
    }
  }
  ```
- `.npmrc`: `node-linker=hoisted`. Expo and React Native's Metro bundler expect a flat `node_modules` layout when used with pnpm, and Expo's monorepo guide recommends this. Setting it now means the web app is tested under the same linker before the phone app arrives.
- `web/package.json`: set `"name": "@ss/web"` and keep its scripts as they are.
- Run `rm -f pnpm-lock.yaml && pnpm install` to regenerate the lockfile at the root.

**2.3 Fix paths that assumed the repo root:**
- `web/vite.config.ts`: the `@` alias uses `path.resolve(__dirname, './src')`, which still works because the file moved with `src/`.
- `web/tsconfig*.json` `paths`, `web/components.json` (shadcn aliases) and `web/eslint.config.js` should all be relative, so check them.
- `.github/workflows/deploy.yml` (build-only; the GitHub Actions deploy is still **on hold**): change it to `pnpm install --frozen-lockfile` + `pnpm --filter @ss/web build`. **Don't add any deploy step.**
- `.gitignore`: make `dist/` and `node_modules/` match at any depth.

**2.4 READMEs:**
- The root `README.md` explains the layout (`web/`, `mobile/`, `desktop/`, `packages/core/`) and the commands.
- The current README content moves to `web/README.md`.

**2.5 Verify:**
- `pnpm install`, then `pnpm build` and `pnpm lint`: both clean.
- Compare `web/dist/` with a build of the pre-move commit (`git stash`/worktree). The file list should be the same and the bundle sizes within a few bytes, apart from hashed names.
- `SS_API_TARGET=https://dev.davidfruin.com pnpm dev`: log in, feed, upload, logout (using a dev test account, per the [[ssreact]] note).
- `git log --follow --oneline web/src/lib/api.ts | head` shows the full history.

**Commit:** `repo: turn ssreact into a pnpm workspace (web/ + future mobile/, desktop/, packages/core)`

**2.6 Done 2026-10-06:** ssreact is deployed to dev's `public_html/app/`.

**2.7 Deploying after the restructure (dev and prod).** The build output moves from `dist/` to `web/dist/`. **Running the old command after the restructure would copy the wrong folder**, so make the new way the only obvious one:
- Add `scripts/deploy-web.sh` to ssreact, taking the target host as its argument (`dev` or `app`):
  ```bash
  #!/usr/bin/env bash
  # Builds web/ and deploys it into <host>.davidfruin.com/public_html/app/. Prod only when Dave says so.
  set -euo pipefail
  HOST="${1:?usage: deploy-web.sh dev|app}"
  case "$HOST" in dev|app) ;; *) echo "unknown host: $HOST" >&2; exit 1;; esac
  if [ "$HOST" = app ]; then read -r -p "Deploy to PROD (app.davidfruin.com)? Type 'prod' to continue: " ok; [ "$ok" = prod ]; fi
  cd "$(dirname "$0")/.."
  pnpm install --frozen-lockfile
  pnpm --filter @ss/web build
  test -f web/dist/index.html   # never rsync --delete from an empty or wrong folder
  rsync -rltz --no-owner --no-group --delete web/dist/ "el1:/home/davidfruin/domains/$HOST.davidfruin.com/public_html/app/"
  ```
- **Rewrite the [[ssreact]] note's Hosting section.** It still describes the retired react.davidfruin.com host and its old exclude-list command. It should describe:
  - dev and prod, both on the split layout;
  - this script;
  - the backend deploy (`ssapi/deploy/`, per [[deploy-layout-plan]] §2);
  - the `v2.0.0` tags as the rollback anchor.
- **Verify:** run `scripts/deploy-web.sh dev` once after the restructure. dev serves the app (login, feed, `/history`), and nothing outside `public_html/app/` changed.

---

## Step 3: create ssterminal from the three C repos (agent, about a day)
**3.0 Dave:** create an empty **public** repo `DavidFruin/ssterminal` (no README, no licence, no .gitignore), and attach it to the agent's session with push access.

**3.1 Import with history.** Use `git filter-repo` (install with `pip install git-filter-repo`). Work on **full** clones, not shallow ones:
```bash
WORK=$(mktemp -d) && cd "$WORK"
git clone https://github.com/DavidFruin/simple-social-cli.git ssterminal       # becomes the root history
git clone https://github.com/DavidFruin/simple-social-cli-interactive.git wiz
git clone https://github.com/DavidFruin/simple-social-tui.git tui
git -C wiz filter-repo --to-subdirectory-filter wizard
git -C tui filter-repo --to-subdirectory-filter tui
cd ssterminal
git remote add wiz ../wiz && git fetch wiz && git merge --allow-unrelated-histories -m "ssterminal: import simple-social-cli-interactive into wizard/" wiz/HEAD
git remote add tui ../tui && git fetch tui && git merge --allow-unrelated-histories -m "ssterminal: import simple-social-tui into tui/" tui/HEAD
git remote remove wiz && git remote remove tui
```
- Use each repo's actual default branch name if it isn't the one `HEAD` points at.
- The CLI repo is already laid out as `lib/`, `vendor/include/`, `cli/`, `tests/`, `Makefile`, `ARCHITECTURE.md`, so it stays at the root.

**3.2 Remove the submodules:**
```bash
git rm wizard/vendor/simple-social-cli tui/vendor/simple-social-cli
git rm wizard/.gitmodules tui/.gitmodules     # or edit them if they list anything else
```

**3.3 One build for everything.** Replace the three Makefiles with **one top-level `Makefile`**:
- builds `lib/libss.a` once;
- builds `simple-social-cli` (from `cli/`), `simple-social-cli-interactive` (from `wizard/src/`) and `simple-social-tui` (from `tui/src/`);
- `make` / `make all` builds all three, and `make cli|wizard|tui` builds one.

Install targets:
- `make install` installs all three, and `make install-cli|install-wizard|install-tui` installs one.
- Keep `PREFIX` (default `/usr/local`) so `make PREFIX=$HOME/.local install` works; one machine already installs to `~/.local/bin`, per the [[simple-social]] note.
- Provide the matching `uninstall` targets.

Carry over from the existing Makefiles **unchanged in substance**, since each is a past bug fix:
- `-MMD -MP` dependency files;
- the static `libss.a`;
- `CURL_LIBS = $(shell pkg-config --libs libcurl 2>/dev/null || echo -l:libcurl.so.4)` (the SONAME fix for the Mint install failures);
- the TUI's `ncursesw` pkg-config fallback;
- each Makefile's `check-lib` intent (a clear error if a dependency is missing).

Include paths become `-Ilib -Ivendor/include` for everything. **Binary names stay exactly the same**, so installed users and their `~/.config/simple-social-cli/config.ini` (shared by all three) notice nothing.

`make test` runs:
- `tests/test_cli.sh`, moved to `cli/tests/` or kept at the root; pick one and update its `cd`;
- the wizard's `wizard/tests/test_wizard.py`.

**3.4 The shared-library version jump (real risk, check carefully):**
- The wizard and the TUI were built against `simple-social-cli` **`d91b04b`** (their submodule pin). The CLI repo's current `HEAD` is **`73431e4`**.
- After the merge, all three build against the **current** `lib/`.
- Run `git diff d91b04b 73431e4 -- lib/` to see what changed. Fix any compile error or behaviour change in the wizard and the TUI. Don't pin an old copy of `lib/`.

**3.5 Verify (Linux, a real build):**
- Install `build-essential` (or equivalent) plus `libcurl` and `ncursesw` dev packages; then `make clean && make`. There should be no warnings beyond today's.
- Run each binary's `--help`, or its first screen.
- Run against the **ssapi local bench** (ssapi plan Phase 0; `php -S`), not dev, so nothing real is touched. Set `TEST_BASE_URL` / `base_url` to the bench:
  - `TEST_EMAIL=… TEST_PASSWORD=… make test`;
  - a manual TUI run in `tmux`: login, feed, post, comment, like, delete, logout.
- `make PREFIX=$(mktemp -d) install` puts three binaries in `bin/`, and `uninstall` removes them.
- `git log --follow --oneline tui/src/app.c | head` shows the TUI's history.

**3.6 Docs:**
- `README.md`: what's in the repo, build and install, the per-OS dependency packages (merge the three READMEs' instructions; keep the LMDE/Debian vs Arch/Omarchy split from the download page), and where the config lives.
- Keep `ARCHITECTURE.md` and the TUI's `notes.md` (move it to `tui/notes.md` if it isn't there already).
- **Install instructions on the website:** ssreact's `web/src/pages/DownloadPage.tsx` currently says "clone each repo with `--recursive`, then `make install`". Change it to "clone `ssterminal`, `make && sudo make install`", then deploy to dev with `scripts/deploy-web.sh dev`; prod only on Dave's go. (simple-social's `download.html` isn't served anywhere since 2026-10-06, so leave it alone.)

**3.7 Push and archive:**
- `git push -u origin` the default branch to `DavidFruin/ssterminal`.
- In each of the three old repos, set the README to:
  > **Archived 2026-10-0X.** Moved to [ssterminal](https://github.com/DavidFruin/ssterminal), full history included. To update an existing install: `git clone https://github.com/DavidFruin/ssterminal.git && cd ssterminal && make && sudo make install`.
- **Dave:** archive the three repos.

**Commits** (in ssterminal):
1. the two import merges;
2. `ssterminal: remove the simple-social-cli submodules`;
3. `ssterminal: one top-level Makefile for all three clients`;
4. fixes from 3.4;
5. `docs: README for the combined repo`.

---

## Step 4: sstests references (agent, about an hour)
1. Attach `sstests` and `grep -rn` it for `simple-social-cli`, `simple-social-cli-interactive`, `simple-social-tui`, `ssreact-native`, `sselectron`, `simple-social/`, `ssreact/src`, `../ssreact` and any hard-coded paths into the other repos.
2. Update them to the new homes:
   - `ssterminal/...`;
   - `ssreact/web/...` for anything that pointed into ssreact.
3. Update its README's "where things live" section.
4. Its Playwright suite targets the **vanilla** frontend on dev. If Step 2.6 puts ssreact on dev, that suite no longer matches dev. Record that in the [[sstests]] note: the suite is now **prod-vanilla-only** until a new ssreact suite exists. Don't delete it.

**Commit:** `tests: update references after repo consolidation`

---

## Step 5: archive simple-social (its condition was met on 2026-10-06)
Prod and dev both run ssapi + ssreact, and the vanilla frontend is retired, so nothing deploys from `simple-social` any more.

**Agent:**
1. Check nothing unique is left:
   - `ARCHITECTURE.md` and `notes.md` stay readable in the archive. Their still-relevant lessons are already summarized in the [[ssreact]] and [[simple-social]] notes. Skim `notes.md` for open wishlist items missing from the [[simple-social]] note, and copy any across.
   - `tests/front-end-test/` is the older 6-spec suite; [[sstests]] already holds the mature one.
   - Tag `v1.0.0` ("Alpha Simple", `28d378f`) must exist on GitHub before archiving. It's recorded in the [[simple-social]] note's Versions section; check with `git ls-remote --tags`.
2. Replace the top of `README.md` with:
   > **Archived 2026-10-0X.** Simple Social 1.x (vanilla web frontend + PHP backend). Since 2.0.0 "Elia" (2026-10-06), the backend lives in [ssapi](https://github.com/DavidFruin/ssapi); the web, phone and desktop apps in [ssreact](https://github.com/DavidFruin/ssreact); the terminal clients in [ssterminal](https://github.com/DavidFruin/ssterminal) (until that exists, the three simple-social-cli/-interactive/-tui repos).

**Dave:** archive the repo.

---

## Step 6: war-table notes (agent, about an hour; do it alongside each step above)
Follow `AGENTS.md`: one note per repo, terse.

**Rename or create these notes:**
- `Projects/ssterminal.md`: **new**, from `Templates/project.md`.
  - Summary: the three clients and the shared `lib/`.
  - Status after Step 3.
  - Decisions: the consolidation, keeping the binary names, and the lib version jump (3.4) with what was fixed.
  - Fold in the still-useful content of `Projects/simple-social-tui.md`: its decisions and gotchas, not its history.
  - There are no separate cli or wizard notes today. The [[simple-social]] note has terminal-client testing details; link to them rather than copying.
- `Projects/ssreact.md`:
  - add a short "Repo layout" section (`web/`, `mobile/`, `desktop/`, `packages/core/`);
  - fold in the **Decisions** bullets from `Projects/ssreact-native.md` and `Projects/sselectron.md`;
  - update the Hosting section after Step 2.6.
- `Projects/ssreact-native.md`, `Projects/sselectron.md` and `Projects/simple-social-tui.md`: set frontmatter `status: archived`, and put a first line "Merged into [[ssreact]] (`mobile/`)" / "(`desktop/`)" / "[[ssterminal]]". Keep the files so old links still work.
- `Projects/simple-social.md`: becomes the **product-level** note (planning, roadmap, decisions that span every repo). Update its repo list to the four repos + "simple-social (archived, 1.x history)".
- `Projects/sstests.md`: Step 4's note.
- `AGENTS.md` → "Machines": on Citadel, repos live under `~/dev`, so the clones become `~/dev/ssapi`, `~/dev/ssreact`, `~/dev/ssterminal` and `~/dev/sstests`. Old clones of archived repos can be deleted once nothing points at them.
- **Plans in `Inbox/`** already describe the new layout (updated 2026-10-03). Agents must use the new paths.

**Commit (war-table):** `repos: consolidate to ssapi/ssreact/ssterminal/sstests; archive merged repos`

---

## Order, estimates, decisions

| Step | Who | Effort | Blocks |
|---|---|---|---|
| 1. Archive the two empty repos | READMEs done; **Dave archives** | 5 min | nothing |
| 2. ssreact workspace (2.0 and 2.6 done) | agent, after the checks before 2.1 | ½–1 day | **phone port Phase 1** |
| 3. ssterminal | Dave creates the repo; agent does the rest; Dave archives | about 1 day | nothing (independent) |
| 4. sstests references | agent | about 1 hour | after 2 and 3 |
| 5. Archive simple-social | agent (README + checks); Dave archives | about 30 min | nothing (prod switched 2026-10-06) |
| 6. Notes | agent | about 1 hour | with each step |

About **2–3 days of agent work** in total, or roughly a week of calendar time on the $20 plan.

**Decisions for Dave:**
1. ~~Merge `deploy-layout` (2.0)~~ and ~~test-deploy ssreact on dev (2.6)~~: both done 2026-10-06.
2. Do the prod real-use check (see "Before 2.1") and clear the stale ssreact row in `Areas/active-work.md`.
3. Permission for an agent to run `gh repo create/archive`, or Dave does those himself.
4. ~~When prod switches~~: done 2026-10-06, so archive `simple-social` (Step 5) whenever convenient.
