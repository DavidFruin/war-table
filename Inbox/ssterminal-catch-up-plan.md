---
status: proposal
written: 2026-10-09
for: Sonnet 5 (medium effort), implementing agent
repos: ssterminal @ f5e86f3 (C: lib/, cli/, wizard/, tui/), ssapi @ acc2873 (reference only), sstests (local bench)
---

# ssterminal catch-up plan: CLI, wizard and TUI

The three terminal apps (`sscli`, the wizard and the TUI) share one C library (`lib/`).

**Where things stand:**
- Their last feature work was **2026-09-24** (the TUI's visual redesign). The 2026-10-06 commits only merged the three repos into [[ssterminal]].
- Since then the server has gained many new actions (moderation, invites, staff roles, replies, sessions, media limits) and changed several old ones. Prod has run all of it since 2026-10-09.
- The library calls **33 of the server's 69 actions** today.

**What's broken or missing for someone using the terminal today:**
- **Can't create an account.** Registration is invite-only and asks for the code first; the terminal never sends a code.
- **Never accepts the Terms**, never sees the minimum age, and never gets the welcome message.
- **Comment replies** show as plain comments, with no threads. Reply notifications show as unknown types.
- **Reported posts don't fold away**, and the author doesn't see the "Hidden by a moderator" label on their own frozen posts.
- **@mentions show as `@user`**, not the person's email. This is an old known gap: the library never reads the `mentions` field.
- **Missing features:** report, block and the blocked list; the invite card; role badges; devices; media limits and storage use.
- **Silent technical risks:**
  - Responses over 64 KB are cut off (`lib/ss_api.c`, `strncpy` into a fixed `API_MAX_RESPONSE` buffer). Feeds now carry more per post (media details, mentions, likes), so a full feed can exceed it.
  - The JSON reader finds a key **anywhere** in the text (`find_key` in `lib/ss_json.c` is a plain `strstr`). Nested objects, like a reply inside a comment, can be read as the outer object's fields.

**Scope.** The terminal apps get everything a member uses on the web, except the following, by default (see the decisions at the end):
- **English only:** the server answers in English when no language header is sent.
- **No staff tools:** freezing, deleting and the Team tab happen only on the website's moderation page ([[staff-roles-plan]] R10). Staff only see badges in the terminal.
- **No push notifications, themes or navigation hand.** Those are web and phone features.

---

## 0. Rules for the implementing agent
- The usual rules apply: read `war-table/AGENTS.md`; check and claim `Areas/active-work.md`; one task per commit; push after each.
- **Never deploy, and never point anything at prod (`app.davidfruin.com`) or dev (`dev.davidfruin.com`).**
  - The library's built-in default server is prod (`lib/ss_config.c`). For all work and tests, use a **local ssapi bench** (`php -S` with a scratch `private/`, as in [[sstests]]) through `config.ini`'s `base_url`, or the `TEST_BASE_URL` that `tests/test_cli.sh` already reads.
- **Order:** library first (all three apps use it), then the CLI (simplest, and scriptable for tests), then the wizard, then the TUI.
- **Every change keeps `--json` output backward-compatible.** New fields are added; existing ones are never renamed or removed. Scripts depend on it.
- **Build cleanly** with `make` (no new warnings), and run `tests/test_cli.sh` against the bench after each phase.
- **TUI changes:** check them in a pty with `tmux send-keys`/`capture-pane` against the bench, as earlier TUI work was verified ([[simple-social-tui]]).

---

## Phase 0: library groundwork (about half a day)
- **0.1 No more cut-off responses.**
  - `api_call()` keeps the full body: return a heap buffer the caller frees, or grow the caller's buffer. Never truncate silently.
  - Raise or remove `API_MAX_RESPONSE`, and have every caller use the new form.
  - **Verify:** a bench feed of 25 posts with 5,000-character text is parsed completely.
- **0.2 Read only the object's own keys.**
  - Make `find_key()` depth-aware: it matches `"key":` only at the top level of the object or array item being read, skipping nested `{…}`/`[…]` and strings.
  - Keep the public function names, so callers don't change.
  - **Verify:** a test (a small C test binary, built by `make test`) where a nested object contains the same key before the real one. Also check that the existing parsers still read the bench's real responses.
- **0.3 Errors carry a code and status.**
  - `api_get_last_status()` (HTTP status) and `api_get_last_code()` (the server's `code`, e.g. `media_quota`), next to the existing `api_get_last_error()`.
- **0.4 Response sizes.** Posts and comments are fixed-size C structs. Add the new fields (Phase 2) with sensible caps, and check the structs aren't so big that a 256-item array on the stack overflows. Move big result arrays to the heap if needed.

**Commits:** `lib: no response truncation`, `lib: depth-aware JSON keys (+ tests)`, `lib: error status and code`.

## Phase 1: joining and first login (about a day)
- **1.1 Code-first registration** (server: [[access-and-public-launch-plan]] Step 1):
  - **Library:**
    - `api_check_invite_code(code, inviter_email_out, …)` calls `checkInviteCode` and returns the inviter's email.
    - `api_register_send_otp(email, invite_code)` gains the code.
  - **Codes are case-sensitive.** Pass them through exactly, URL-encoded (they contain `! $ & @ ?`).
  - **CLI:** `register` becomes `register <code> <email> <password> <confirm>`. It checks the code first and prints "Invited by …", then asks the age and terms question (1.2), sends the email code, and prompts for it, or takes `--otp` as today.
    - `--json` keeps working for scripts: an `--agree` flag answers the age and terms question.
  - **Wizard and TUI:** an invite-code step first; the email field appears only after the code is accepted, as on the web.
- **1.2 Age and terms:**
  - **Registration** requires answering "I agree to the Terms of Use (https://app.davidfruin.com/terms) and I'm at least 13 years old" with yes.
  - **After every login**, read `termsVersionAccepted`/`termsVersionCurrent` from `getMyInfo`. When the accepted version is older:
    - **TUI and wizard:** show the terms link and ask to accept; accepting calls `acceptTerms(version)`, declining logs out.
    - **CLI:** print a one-line notice on stderr on every command until accepted, and add an `accept-terms` command.
  - The server doesn't enforce this; the apps do, as the web and phone do.
- **1.3 Welcome:** when `getMyInfo` has `welcome` (`{inviterEmail, inviterIsOwner}`), show the welcome text once, using the same wording as the web (access plan §1.1). Then call `dismissWelcome`. The CLI prints it once, on the first command after login.
- **1.4 Frozen accounts:** login is refused with the server's message. Show it as is.
- **Verify on the bench:**
  - Register with a code generated by a bench member (wrong code, then right code). The inviter is shown, an empty age/terms answer is refused, and the account is created.
  - The terms prompt appears after a terms-version bump, and the welcome shows once.
  - `tests/test_cli.sh` gains these cases.

**Commits:** `lib: invite check, invite code on register, terms, welcome`, `cli: code-first register, terms and welcome`, `wizard: …`, `tui: …`.

## Phase 2: the content people see (about 1.5–2 days)
- **2.1 Posts:**
  - Parse these new fields:
    - `commentCount`, `reportedByMe`, `frozen` (only ever on your own content);
    - `mentions` (`[{id, email}]`);
    - `media` (`{type, url, variantUrl, posterUrl, width, height, duration, loop}`), keeping `mediaUrl`.
  - Show mentions as the real email (fixes the old `@user` gap). Show media as its type and URL, and a looping GIF as "GIF (plays as a loop)".
- **2.2 Fold-away** (access plan Step 1C): a post or comment with `reportedByMe` shows folded.
  - **TUI:** one line, "You reported this post." with `s` to show. It folds again when you leave the screen.
  - **CLI and wizard:** the line, plus the text only with `--show-reported`.
- **2.3 Your own frozen content** shows "Hidden by a moderator", with no like or comment actions.
- **2.4 Replies** (ssapi `acc2873`; one level deep):
  - **Library:** `api_get_post_comments(…, threaded)` reads top-level comments with their `replies` and reply counts. `api_create_comment(post_id, text, parent_id)` gains the parent.
  - **Placeholders:** "Comment deleted" for deleted comments, and "frozen by staff" for frozen ones, as the server sends them.
  - **CLI:** `comments <postId>` shows the threads indented. New command `reply <postId> <commentId> <text>`.
  - **TUI:** the post detail view shows replies indented under their comment, and `r` replies to the selected comment.
  - **Wizard:** a Reply menu entry.
  - **Verify** that the depth-aware reader (0.2) is what keeps a reply's fields out of its parent.
- **2.5 Notifications:**
  - Text for every type the server sends: `like`, `unlike`, `comment`, `reply` ("replied to your comment"), `thread_reply` ("also replied in a thread"), `follow`, `unfollow`, `mention`.
  - `comment_id` takes you to that comment in the TUI.
  - Anything unknown shows a neutral "did something" line rather than nothing.
- **2.6 Badges:** `getUserInfo` now has `role`. Profiles show User, Moderator, Admin or Owner. Optionally, mark staff in lists using `getStaff`.
- **2.7 Word filter:** posting or commenting can be refused with "…contains a word that isn't allowed". Show the message, and in the TUI keep the text in the editor so it can be fixed.

**Commits:** one per item, library then each app.

## Phase 3: new member features (about 1.5 days)
- **3.1 Report and block** (server: Step 1B):
  - **Library:** `api_report(target_type, target_id, reason, details)`, `api_block_user`, `api_unblock_user`, `api_get_blocked_users`.
  - **Reasons:** exactly the server's list: `spam`, `harassment`, `hate`, `sexual`, `violence`, `self_harm`, `illegal`, `other`.
  - **CLI:** `report post|comment|user <id> <reason> [details]`, `block <userId>`, `unblock <userId>`, `blocked`.
  - **TUI:** a report key in the post, comment and profile views (a reason menu, then a confirmation), a block option on profiles, and a Blocked list in Settings.
  - **Wizard:** the same as menu entries.
  - After reporting, the item folds away (2.2).
- **3.2 Invites:**
  - **Library:** `api_get_my_invites`, `api_generate_invite_code`, `api_cancel_invite_code`.
  - **CLI:** `invite` shows your invite state or live code; `invite --generate` and `invite --cancel <code>`.
  - **The note**, shown before generating: **"You can only invite one person, so choose wisely!"**
  - **The owner** sees all live codes and can generate more.
  - **TUI:** an "Invite someone" section on the Me tab, showing the code and its expiry (people can select and copy it from the terminal).
- **3.3 Devices:**
  - **Library:** `api_get_sessions`, `api_revoke_session(id)`, `api_revoke_all_other_sessions`.
  - **CLI:** `devices`, `devices --revoke <id>`, `devices --revoke-others`.
  - **TUI:** a Devices section in Settings.
  - The device name comes from the user agent the library already sends.
- **3.4 Media rules:**
  - Call `getMediaLimits` before an upload: size per type, `videoSupported`/`audioSupported`, and storage used/limit. Refuse early with the same wording as the web.
  - **On the server's answers:**
    - **413 `media_quota`:** show the exact message ("You have reached your media storage limit of 1 GB of media.").
    - **429:** "Too many uploads".
    - **503:** retry once after 3 seconds, then show the message.
  - **Accepted types** are the server's list from [[media-pipeline-plan]] A5 (the file picker can filter on it).
  - Show storage use in the TUI's Settings and in `whoami`/`profile`.

**Commits:** one per feature, library then each app.

## Phase 4: docs, tests and wrap-up (about half a day)
- **Update the docs:** `README.md`, every `--help` text, the wizard's and TUI's help screens, and `ARCHITECTURE.md`. The latter is stale: it still talks about the `simple-social-cli` repo, "40 endpoints", "28 commands" and `users.jwt`.
- **`tests/test_cli.sh`:** a case for each new command, run against the local bench. Write down in the script's header how to start the bench.
- **The website's Download page** (ssreact `web/src/content/download.*.tsx`): check the terminal section still matches (registration now needs an invite code). Make any change as a small ssreact commit, in English and Spanish.
- **Notes:** record in war-table's [[ssterminal]] note what changed, what was verified live in a pty, and what wasn't.

---

## Estimate and decisions

| Phase | Work | Effort |
|---|---|---|
| 0 | Library robustness (no truncation, depth-aware JSON, error codes) | about half a day |
| 1 | Code-first registration, terms and age, welcome | about a day |
| 2 | New post and comment fields, fold-away, replies, notifications, badges, word filter | 1.5–2 days |
| 3 | Report and block, invites, devices, media rules | about 1.5 days |
| 4 | Docs, tests, Download page | about half a day |

About **5 days of agent work**. Phases 0–1 alone (about 1.5 days) fix the worst problem: nobody can join from the terminal.

**Decisions for Dave (defaults in brackets):**
1. The terminal stays English-only [yes].
2. No staff tools in the terminal: moderation stays on the website, and staff only see badges [yes].
3. All three apps get the features [yes]. The alternative is giving the wizard only Phases 0–1, since the TUI and CLI cover the rest.
4. The terminal asks for terms acceptance after login, as the web and phone do [yes].
