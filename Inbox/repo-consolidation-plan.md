---
status: proposal
written: 2026-10-03
for: Sonnet 5 (medium effort), implementing agent; GitHub settings steps for Dave
repos: ssreact @ 865ff6f (+ branch deploy-layout @ 466367f), simple-social-cli @ 73431e4, simple-social-cli-interactive, simple-social-tui, ssreact-native, sselectron, simple-social, sstests, ssapi
---

# Repo consolidation plan: four Simple Social repos

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
| `simple-social` | **Frozen now, archived later.** It still holds the old vanilla web frontend, which is what `app.davidfruin.com` (prod) and `dev.davidfruin.com` (in `public_html/app/`) serve today. Its backend copy is already obsolete (dev runs ssapi; prod switches when Dave says so). | Archived once prod serves ssreact + ssapi (Step 5) |

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

**2.0 Merge the waiting L4 branch first.**
- `deploy-layout` (`466367f`) drops `public/.htaccess`.
- It was meant to merge "on react.davidfruin.com's migration day". That host is being retired, and every future ssreact host uses the split layout, whose root `.htaccess` comes from `ssapi/deploy/`.
- So merge it into `master` now (Dave confirms). Do it before the restructure, so the merge isn't fighting moved paths.

**2.1 Move the app into `web/`, keeping history:**
```bash
git checkout master && git pull
mkdir web
git mv src public index.html vite.config.ts tsconfig*.json eslint.config.js components.json package.json web/
# Leave at the root: .git*, .github/, README.md, pnpm-lock.yaml (regenerated below)
```
Check `git ls-files` for anything else that belongs to the app, such as `postcss` configs or `.env.example`, and move it too.

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

**2.6 How ssreact gets deployed now** (it has no live host since react.davidfruin.com is being retired):
- **DECISION for Dave:** recommended is to deploy ssreact to **dev**'s `public_html/app/`, replacing the vanilla frontend there. dev is a test host, already on the split layout, already running ssapi.
- Command:
  ```bash
  pnpm build && rsync -rltz --no-owner --no-group --delete web/dist/ el1:/home/davidfruin/domains/dev.davidfruin.com/public_html/app/
  ```
- After that, the vanilla frontend lives only on prod until prod switches (Step 5).

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
- **Install instructions on the websites:** they currently say "clone each repo with `--recursive`, then `make install`". Update them to "clone `ssterminal`, `make && sudo make install`":
  - ssreact: `web/src/pages/DownloadPage.tsx`;
  - simple-social: `download.html`. It's still served by prod, so this is a one-line-scope commit that Dave deploys with his usual process.

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

## Step 5: simple-social, frozen now and archived later
**Now (agent):** add a banner to the top of `simple-social`'s README:
> **Frozen 2026-10-03.**
> - The backend lives in [ssapi](https://github.com/DavidFruin/ssapi).
> - The web, phone and desktop apps live in [ssreact](https://github.com/DavidFruin/ssreact).
> - The terminal clients live in [ssterminal](https://github.com/DavidFruin/ssterminal).
>
> This repo only keeps the legacy vanilla web frontend that `app.davidfruin.com` still serves, until ssreact replaces it. No new features here; only fixes Dave asks for.

**Later (Dave decides when):** once prod (`app.davidfruin.com`) runs **ssapi** (deploy-layout migration + the security/speed fixes) **and serves ssreact** from `public_html/app/`:
1. Check nothing unique is left:
   - `ARCHITECTURE.md` and `notes.md` stay readable in the archive. Their still-relevant lessons are already summarized in the [[ssreact]] and [[simple-social]] notes.
   - `tests/front-end-test/` is the older 6-spec suite; [[sstests]] already holds the mature one.
2. Update its README to "Archived".
3. **Dave:** archive the repo.

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
- `Projects/simple-social.md`: becomes the **product-level** note (planning, roadmap, decisions that span every repo). Update its repo list to the four repos + "simple-social (frozen, legacy vanilla frontend)".
- `Projects/sstests.md`: Step 4's note.
- `AGENTS.md` → "Machines": on Citadel, repos live under `~/dev`, so the clones become `~/dev/ssapi`, `~/dev/ssreact`, `~/dev/ssterminal` and `~/dev/sstests`. Old clones of archived repos can be deleted once nothing points at them.
- **Plans in `Inbox/`** already describe the new layout (updated 2026-10-03). Agents must use the new paths.

**Commit (war-table):** `repos: consolidate to ssapi/ssreact/ssterminal/sstests; archive merged repos`

---

## Order, estimates, decisions

| Step | Who | Effort | Blocks |
|---|---|---|---|
| 1. Archive the two empty repos | Dave (+ agent for READMEs) | 15 min | nothing |
| 2. ssreact workspace | agent; Dave confirms 2.0 and decides 2.6 | ½–1 day | **phone port Phase 1** |
| 3. ssterminal | Dave creates the repo; agent does the rest; Dave archives | about 1 day | nothing (independent) |
| 4. sstests references | agent | about 1 hour | after 2 and 3 |
| 5. simple-social freeze → archive | agent (banner); Dave (archive later) | minutes / later | prod switch |
| 6. Notes | agent | about 1 hour | with each step |

About **2–3 days of agent work** in total, or roughly a week of calendar time on the $20 plan.

**Decisions for Dave:**
1. Confirm merging the `deploy-layout` branch (2.0).
2. Where ssreact is test-deployed now that react.davidfruin.com is retired (2.6). dev is recommended.
3. Permission for an agent to run `gh repo create/archive`, or Dave does those himself.
4. When prod switches to ssapi + ssreact, which is what lets `simple-social` be archived.
