---
status: proposal
written: 2026-10-28
for: Sonnet 5 (medium effort), implementing agent
repos: ssreact @ e70cc01 (packages/core, web/, mobile/), ssapi @ acc2873 (deploy files only), sstests
---

# Share links plan: members-only links to posts, comments and profiles

Dave (2026-10-28) chose **members-only** links ("I want only member visible links") over public share links. A link sent to anyone opens the post, comment or profile **for a logged-in member**. Everyone else sees the login page with a short note, and **nothing about the content**.

**What already exists:**
- Every post and profile has an address: `/post/<postId>` (post ids look like `<userId>.<unixTime>`) and `/profile/<userId>`.
- Comments are linked as `/post/<postId>?comment=<commentId>`. Reply notifications already use this; the web post page and the phone post screen (`useLocalSearchParams` `comment`) jump to the comment and highlight it.
- Logged-out visitors are sent to `/login` and back afterwards (`web/src/routes/RequireAuth.tsx` + `LoginPage`). **But only the path survives** (`from?.pathname`), so `?comment=` is lost.
- The server already requires login for all content. Nothing changes on the server's access rules.

**What's missing (this plan):**
- No share or copy button anywhere except the invite card.
- Phones open these links in the browser, not the app.
- Pasted links show no preview.
- A missing, hidden or deleted post gives an error message, not a friendly page.

**Privacy rules for every part of this plan:**
- **No content leaves the app through a link.** The preview, the login note and the page title are generic for every link: no post text, names or emails.
- **Same visibility as in the app.** Blocked people, frozen and deleted content, and fold-away reported content all behave exactly as they do now. A link never shows a member more than they could see by browsing.
- **Search engines** are told not to list post and profile pages.

---

## 0. Rules for the implementing agent
- The usual rules apply: read `war-table/AGENTS.md`; check and claim `Areas/active-work.md`; one task per commit, pushed after each.
- **Never deploy, and never touch prod (`app.davidfruin.com`) or dev (`dev.davidfruin.com`).** Test on a local bench.
- **All new text** goes through `tr()`, with Spanish in `packages/core/src/i18n/es.ts`. The web lint rule enforces it.
- **Mobile:** follow `mobile/AGENTS.md` and read the Expo SDK 57 docs before touching app config or linking.
- Phase 3 needs two values from Dave and a new phone build, so stop before it if he hasn't given them (see 3.0).

---

## Phase 1: web (about half a day)
### 1.1 One link helper, in `@ss/core`
`shareUrl(base, target)` returns the canonical link, with the ids URL-encoded:
- `https://<host>/post/<postId>`
- `https://<host>/post/<postId>?comment=<commentId>`
- `https://<host>/profile/<userId>`

`base` is the site the app talks to (`window.location.origin` on the web, `apiBase` on the phone), so dev builds share dev links.

Add a test for each shape, including a post id with a dot.

### 1.2 Share and copy buttons
- **Posts** (`web/src/components/post/PostCard.tsx`, the same menu that has Report): **Copy link**, plus **Share…** when `navigator.share` exists.
- **Comments and replies** (`CommentItem.tsx`): the same pair, with the comment link.
- **Profiles** (`ProfilePage.tsx`): the same pair, with the profile link. On your own profile too.
- **Copying** shows a short toast, "Link copied". If the clipboard is blocked (an insecure context or a refused permission), fall back to a small dialog with the link selected.
- **Share text** is just the link and a generic line ("A post on Simple Social"), never the post text.
- **What's left out:**
  - Your own frozen content doesn't get these options.
  - Fold-away reported content gets them only after Show.

### 1.3 Logged-out visitors
- **Keep the whole target.** In `RequireAuth.tsx`/`LoginPage.tsx`, keep `pathname + search + hash` in `from`, and return to all of it after login. That fixes the lost `?comment=`.
- **A short note on the login page** when `from` is a post, comment or profile: "Someone shared this with you on Simple Social, a private, invite-only network. Log in to see it. If you don't have an account, ask the person who sent the link for an invite." No content, names or ids.
- **Register keeps the target too.** It leads to the same place after the account is created and the first login.

### 1.4 Not available
When a linked post, comment or profile can't be shown (deleted, frozen, hidden by a block, or a bad id), show a calm page instead of the raw error: "This isn't available. It may have been deleted, or you may not have access to it." with a button to the feed.
- Don't say which of those it was; the server already doesn't.
- **A comment link whose comment is gone** still opens the post, with a small note: "That comment isn't available."

### 1.5 Generic link previews and search engines
- **`web/index.html` gets static Open Graph tags,** the same for every address:
  - `og:title` "Simple Social"
  - `og:description` "A private, invite-only network. Log in to see what was shared."
  - `og:image`: the app icon from `web/public/pwa-icons/` as an absolute https URL. Check the deployed path.
  - `og:type` website
  - `twitter:card` summary

  The site is a single-page app, so these are what WhatsApp, iMessage and others show. Being generic is the point.
- **Search engines:** in `ssapi/deploy/root.htaccess` (the one canonical web-root file), send `X-Robots-Tag: noindex` for `/post/` and `/profile/` addresses. This is a small ssapi commit; Dave deploys it with the web root file.

### 1.6 Verify (`pnpm dev` against the bench, desktop and 375 px)
- **Copy link** on a post, a reply and a profile gives the right addresses, and the toast appears. **Share…** appears only where the browser supports it.
- **Logged out:**
  - A post link goes to login, shows the note, and lands on the post after login.
  - A comment link lands on the post with that comment highlighted.
  - A profile link lands on the profile.
- **Links that can't be shown:**
  - A link to a deleted post shows the "isn't available" page.
  - So does a post by someone who blocked you.
  - A link to a deleted comment opens the post with the note.
- **Previews and headers:**
  - The page source has the generic OG tags.
  - `curl -I` on the bench's Apache for `/post/x` shows the `noindex` header. The deploy-layout bench (real Apache) is needed for headers.
- `pnpm typecheck`, `pnpm lint`, `pnpm test` and `pnpm build` pass.

**Commits:** `core: share link helper`, `web: copy and share links`, `web: return to the full link after login, with a note`, `web: friendly not-available page`, `web: generic link previews`, `deploy: noindex for post and profile pages` (ssapi).

---

## Phase 2: phone, JavaScript only (about half a day)
- **Share:** a **Share** action in the menus of `mobile/src/components/PostCard.tsx`, `CommentItem.tsx` and `ProfileView.tsx`. It opens React Native's `Share` sheet with `shareUrl(apiBase, …)`, and the sheet has Copy built in. Follow the same rules as 1.2 for frozen and folded content.
- **Opening links in the app** (works once Phase 3 makes the phone treat the links as the app's own; until then it's testable with the `simplesocial://` scheme):
  - Expo Router maps `/post/<id>` and `/profile/<id>` to the existing `(tabs)/post/[id]` and `(tabs)/profile/[id]` screens. The post screen already reads `comment`.
  - **Check** that a link opened while logged out goes to login and then to the target, keeping `comment`. Fix it if it doesn't, using the same rule as 1.3.
  - **Check** that a link opened while the app is already running replaces the current screen sensibly (Back returns to where you were).
- **Not available:** the same calm message as 1.4 on the post and profile screens.
- **Verify:**
  - `npx expo lint` and `npx tsc --noEmit` pass.
  - On a development build or Expo Go: the share sheet opens with the right link.
  - `npx uri-scheme open "simplesocial://post/<id>?comment=<cid>" --android` (and `--ios`) opens the post at that comment, logged in and logged out. Check the exact command in the SDK 57 linking docs.

**Commits:** `mobile: share links`, `mobile: open shared links (logged out too)`, `mobile: not-available message`.

---

## Phase 3: phones open the links in the app (with the phone release; about half a day, plus Dave's two values)
For a tapped `https://app.davidfruin.com/post/…` link to open the app instead of the browser, the phone checks a file on the website that names the app. This needs a **new native build**; an over-the-air update isn't enough. So it ships with the phone release ([[ssreact-native-port-plan]] Phase 8).

### 3.0 From Dave (neither value is secret)
1. **Apple Team ID:** in the Apple Developer account, under Membership details.
2. **The Android app's signing certificate SHA-256 fingerprint:** from `eas credentials` → Android, after the first Android build creates the key.

### 3.1 App config (`mobile/app.config.ts`)
- **iOS:** `associatedDomains: ['applinks:app.davidfruin.com']`.
- **Android:** an `intentFilters` entry with `autoVerify: true` for `https://app.davidfruin.com` with path prefixes `/post/` and `/profile/`.
- Check both keys against the SDK 57 docs.
- Dev builds may also list `dev.davidfruin.com`. That's only worth it if Dave tests links on dev.

### 3.2 The two files on the website (in ssapi `deploy/public/.well-known/`, deployed with the web root)
- **`apple-app-site-association`** (no extension, served as `application/json`): `applinks` for `<TeamID>.com.davidfruin.simplesocial`, covering `/post/*` and `/profile/*`.
- **`assetlinks.json`:** `delegate_permission/common.handle_all_urls` for package `com.davidfruin.simplesocial` with the SHA-256 fingerprint.
- **`root.htaccess`** must serve `/.well-known/` as real files, not the single-page app, with the right content types and no redirects (Apple refuses redirects).

### 3.3 Verify (Dave's phones, after the family builds)
- Tapping a post link in Messages or WhatsApp opens the app at the post. A comment link opens it at the comment. A profile link opens the profile.
- With the app not installed, the same link opens the website, and that still works.
- Record the results in [[ssreact]].

**Commits:** `mobile: open app.davidfruin.com links in the app`, `deploy: app link files` (ssapi).

---

## Not in this plan
- **Public links for non-members:** Dave chose members-only.
- **Previews with the post's own text or photo:** these would leak content to messaging apps' servers.
- **Terminal apps:** `sscli post <id>` already takes an id. Printing each post's link, or accepting a pasted link, can be added to [[ssterminal-catch-up-plan]] later if wanted.

## Estimate
| Phase | Work | Effort |
|---|---|---|
| 1 | Web: helper, share and copy buttons, return to the full link, login note, not-available page, generic previews, noindex | about half a day |
| 2 | Phone: share sheet, opening links (logged out too), not-available | about half a day |
| 3 | Phones open links in the app: config and two website files | about half a day, after Dave's two values and with the phone release |

Phases 1 and 2 can ship any time. Phase 3 rides on the first phone build.
