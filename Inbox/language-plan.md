---
status: proposal
written: 2026-10-08
for: Sonnet 5 (medium effort), implementing agent
repos: ssreact @ 658f494 (packages/core, web/, mobile/), ssapi @ 768fe86, sstests
---

# Language plan: English and Spanish, with a picker in Settings

Dave (2026-10-08): "add [a] language picker to the settings of the app. I only need Spanish and English for at least a year or so."

The goal: every screen of the web app and the phone app, everything the server says to people (error messages, push notifications, emails) and the pages family members need (landing, download, terms, privacy, conduct) are available in **English and Spanish**. The language is chosen in **Settings**, follows the person's account across devices, and defaults to the phone's or browser's language.

**Scope is two languages, on purpose.** Don't build anything for a third (no language-detection services, no translation platform, no right-to-left support). The design below still makes a third language cheap later: one more dictionary file in `@ss/core` and one in ssapi.

**Not translated:**
- What people write (posts, comments). There's no automatic translation.
- Email addresses, which are the names shown in the app.
- The API docs, Roadmap and History pages: developer-facing and they change often. In Spanish mode they show one line at the top: "Esta página solo está disponible en inglés."
- The Admin page and the moderation report emails, which only Dave sees.
- ssterminal (CLI/TUI): it stays English. The server's default stays English, so those clients see no change.
- Timestamps. The app shows `YYYY-MM-DD HH:MM:SS`, which works in both languages. Relative times ("3 minutes ago") are translated.

---

## 0. Rules for the implementing agent
- The usual rules apply: check and claim `Areas/active-work.md`, one task per commit, push after each commit, **never deploy**, never touch prod. Dave deploys.
- Tests go in [[sstests]] or `packages/core/src/__tests__` (the existing Vitest suite), never in ssapi.
- **Mobile:** follow `mobile/AGENTS.md`. Read the Expo **SDK 57** docs before using any Expo API (`expo-localization`, the `locales` app config), and don't work from memory. Install with `npx expo install`.
- The ssapi database change uses the **next free `migrationN`** (SCHEMA_VERSION is 4 as of `768fe86`, so most likely `migration5`). Check first, and don't run in parallel with another agent adding a migration.
- Skip anything marked **DECISION** until Dave answers. Use the stated default if he says "go" without answering.

---

## 1. Design decisions (read before coding)

**L1. Our own small typed dictionaries in `@ss/core`, not i18next.**
- With two languages, about 100 lines of TypeScript do the job.
- `@ss/core` has no React dependency, and the web and phone apps run different React versions (see [[ssreact-native-port-plan]]). Pure functions in core plus one small React provider per app avoid putting a React library in core.
- TypeScript checks completeness: if a Spanish string is missing, typecheck fails, so nothing ships half-translated.

**L2. The translate function is called `tr`, never `t`.** The phone app already uses `const { t } = useTheme()` for theme colours in 31 files, so `t` would collide.

**L3. Which language is shown:**
1. A `?lang=es` or `?lang=en` URL parameter. The website saves it, so a page the phone app opens in the browser matches the app.
2. Logged in: the account's language (`users.lang` on the server), so every device agrees.
3. Logged out: this device's saved choice (`ss_lang`).
4. Otherwise, the phone's or browser's language list: the first entry that is `es*` or `en*`. If neither appears, English.

**Syncing with the account, without surprises:**
- **Changing the language while logged in** saves it on the server straight away (optimistic, reverted on error, like the theme).
- **Changing it while logged out** (on the login page) also sets `ss_lang_pending`. At the next login, the pending choice is sent to the server and the flag is cleared. So someone who switches to Español on the login screen stays in Español after logging in, even if the account said English.
- **Otherwise, at login or app start:** if the account has a language, the app switches to it. If it has none (`NULL`: every existing account), the app sends its current language up, so push notifications and emails match from then on.

**L4. The server only translates when asked.** The apps send `X-SS-Lang: en|es` on every request. With no header the server answers in English, exactly as today. That protects the CLI/TUI and any old cached web build. The standard `Accept-Language` header isn't used, because browsers send it automatically, and an old English build in a Spanish browser would start getting Spanish errors.

**L5. On the server, the English text is the key.** `tr('Invalid email')` looks the sentence up in `src/I18n/es.php` (`'Invalid email' => 'Correo electrónico no válido'`) and falls back to the English text. `respond()` translates the `message` field itself, so the roughly 150 existing `bad('…')` calls with literal text need **no changes**. Only messages built from variables are rewritten with `{placeholders}`.

**L6. Push notifications use the recipient's language, and emails use the request's.** A push goes to another person, so it uses their stored `users.lang`. The one-time-code emails go to whoever is looking at the app right now, so they use `X-SS-Lang`.

**L7. Plurals and relative times are written by hand.** English and Spanish both use the singular only for exactly 1, so each plural is a `.one`/`.other` pair. Relative times ("hace 3 minutos") come from the same dictionaries rather than `Intl.RelativeTimeFormat`, because Hermes, the phone's JavaScript engine, doesn't reliably support it on every platform.

**L8. Spanish style (DECISION, with defaults).**
- **Variety:** neutral Latin American Spanish is the default. The alternative is Spain's Spanish (e.g. "móvil", "ordenador").
- **Tone:** "tú", not "usted".
- **No gendered words about the user,** since the app doesn't know anyone's gender. Write "Te damos la bienvenida", not "Bienvenido".
- **Sentence case for headings,** as normal in Spanish: "Configuración de la cuenta", not "Configuración De La Cuenta".
- **Punctuation and accents:** opening ¿ and ¡, and correct accents always.
- **Glossary** (use these consistently; the reviewer may change them):

| English | Spanish |
|---|---|
| post | publicación |
| like (noun) / liked | me gusta / le dio me gusta |
| comment | comentario |
| follow / follower(s) | seguir / seguidor(es) |
| feed | Inicio |
| notifications | notificaciones |
| settings | Configuración |
| log in / log out | iniciar sesión / cerrar sesión |
| register | crear cuenta |
| mention | mencionar / mención |
| block | bloquear |
| report | reportar (Spain: denunciar) |
| device | dispositivo |
| code (one-time) | código |
| draft | borrador |
| storage | almacenamiento |
| theme | tema |
| upload | subir |
| Terms of Use | Términos de uso |
| Privacy Policy | Política de privacidad |
| Code of Conduct | Código de conducta |

---

## 2. Phase 1: the shared dictionaries in `@ss/core` (about half a day)
**Done 2026-10-08 (local only, not deployed):** ssreact `926d002` (dictionaries + translate), `d322525` (shared text takes a language), plus the API-client header commit. Deviation: the `lang` parameters default to `'en'` so existing web and phone callers keep compiling; Phases 3/4 pass the real language. Dictionaries hold only the core strings so far (time, notifications, report reasons, storage); each screen's strings are added as it is converted.

### 1.1 `packages/core/src/i18n/`
```ts
// index.ts
import { en } from './en';
import { es } from './es';

export type Lang = 'en' | 'es';
// Each language's name in its own language, so anyone can find theirs.
export const LANGS: readonly { value: Lang; name: string }[] = [
  { value: 'en', name: 'English' },
  { value: 'es', name: 'Español' },
];
export const LANG_HEADER = 'X-SS-Lang';
export const LANG_KEY = 'ss_lang';                  // this device's language
export const LANG_PENDING_KEY = 'ss_lang_pending';  // chosen while logged out, sent at next login

export type MessageKey = keyof typeof en;
type PluralBase = { [K in MessageKey]: K extends `${infer B}.one` ? B : never }[MessageKey];
type Params = Record<string, string | number>;

const DICTS: Record<Lang, Record<MessageKey, string>> = { en, es };

export function isLang(value: unknown): value is Lang {
  return value === 'en' || value === 'es';
}

// First es*/en* entry in a preference list (navigator.languages, or
// expo-localization's language tags); English when neither appears.
export function langFromTags(tags: readonly string[]): Lang {
  for (const tag of tags) {
    const base = tag.toLowerCase().split(/[-_]/)[0];
    if (isLang(base)) return base;
  }
  return 'en';
}

export function translate(lang: Lang, key: MessageKey, params?: Params): string {
  const text = DICTS[lang][key] ?? en[key];
  if (!params) return text;
  return text.replace(/\{(\w+)\}/g, (match, name: string) => (name in params ? String(params[name]) : match));
}

// English and Spanish both use the singular only for exactly 1.
export function translatePlural(lang: Lang, base: PluralBase, count: number, params?: Params): string {
  return translate(lang, `${base}.${count === 1 ? 'one' : 'other'}` as MessageKey, { count, ...params });
}
```
- **`en.ts`:** `export const en = { … } satisfies Record<string, string>;`, with flat dotted keys grouped by screen (`'settings.language.title'`, `'post.comments.one'`, `'error.network'`) and a comment header per group.
- **`es.ts`:** `export const es: Record<keyof typeof en, string> = { … };`, in the **same order** as `en.ts` so a reviewer can read them side by side. A missing or extra key is a type error.
- Export it all from `packages/core/src/index.ts`.

### 1.2 Move text that's in core today onto the dictionaries
`grep` core for English strings. Every function that produces text gets a `lang: Lang` parameter (core stays React-free):
- **`format.ts`:**
  - `relativeTime(timestamp, lang)` uses `time.justNow` and the `time.minutesAgo` / `hoursAgo` / `daysAgo` / `weeksAgo` `.one`/`.other` pairs, plus `time.unknown`.
  - `displayEmail(email, currentUserEmail, lang)`: `(you)` becomes `(tú)`, and the `User` fallback is translated too.
- **`moderation.ts`:** `REPORT_REASONS` entries get a `labelKey: MessageKey` instead of `label`, and `reportReasonLabel(reason, lang)` uses it.
- Whatever text is in `media-storage.ts` (`storageUsageText`), `media-limits.ts`, `session-expiry.ts` and `toast.ts`.
- **New `notificationText(n, currentUserEmail, lang)`** in core. The web (`NotificationsPage.tsx` `getText`) and phone notification lists each have their own copy today. Both switch to this one.

### 1.3 The API client sends the language
`ApiConfig` gains `getLang?: () => Lang`. In every request path of `api-client.ts`, the three places that build headers (`call`, the `refreshToken` request and `mediaRequest`), add `[LANG_HEADER]: this.config.getLang()` when it's set. `grep` web/ and mobile/ for any `fetch(` or `XMLHttpRequest` outside the client (e.g. upload progress) and add the header there too.

### 1.4 Tests (`packages/core/src/__tests__/i18n.test.ts`)
- Every key's `{placeholders}` are the same set in `en` and `es`.
- No empty strings.
- Every `.one` has an `.other` in both languages.
- **Untranslated copies:** fail when an `es` value is identical to the `en` one, except keys on an explicit `SAME_IN_BOTH` allow-list (brand names, "Email", "OK", "Hacker", "API").
- `translate` substitutes parameters and leaves unknown `{x}` alone.
- `langFromTags`: `['es-MX']` → es, `['fr-FR','es-ES']` → es, `['de']` → en, `[]` → en.
- `relativeTime` in both languages, at 1 and many for each unit.

**Commits:** `core: i18n dictionaries + translate`, `core: shared text takes a language`, `core: API client sends X-SS-Lang`.

---

## 3. Phase 2: server (ssapi, about half a day to a day)
**Done 2026-10-08 (local only, not deployed):** ssapi `10405fd` (users.lang + updateLanguage, migration5), `7427f1c` (tr() and 131 Spanish messages), `5680cf8` (push and emails), ssreact `8c24162` (API docs), sstests `1453883` and a follow-up (bench checks + message checker). Details and deviations in [[ssapi]].

### 2.1 Migration: `users.lang`
In the next free `migrationN`: `ALTER TABLE users ADD COLUMN lang TEXT` (NULL means "never chosen") and bump `SCHEMA_VERSION`. Don't edit migrations that already shipped.

### 2.2 `src/I18n/i18n.php` + `src/I18n/es.php`
Add `src/I18n/i18n.php` to `composer.json`'s `autoload.files` (both `api.php` and `media.php` load it through `vendor/autoload.php`), then run `composer dump-autoload`.
```php
<?php
// English is the source language: the English sentence is the key, and
// es.php maps it to Spanish. Anything not in es.php comes back unchanged.
const SUPPORTED_LANGS = ['en', 'es'];

// The language the app asked for with X-SS-Lang, or null when it didn't
// (the CLI/TUI and old app builds), which means English.
function explicitRequestLang(): ?string {
    $lang = strtolower(trim($_SERVER['HTTP_X_SS_LANG'] ?? ''));
    return in_array($lang, SUPPORTED_LANGS, true) ? $lang : null;
}

function requestLang(): string {
    return explicitRequestLang() ?? 'en';
}

function tr(string $text, array $params = [], ?string $lang = null): string {
    static $es = null;
    if (($lang ?? requestLang()) === 'es') {
        $es ??= require __DIR__ . '/es.php';
        $text = $es[$text] ?? $text;
    }
    foreach ($params as $name => $value) $text = str_replace('{' . $name . '}', (string)$value, $text);
    return $text;
}
```
`es.php` returns `['English sentence' => 'Frase en español', …]`, grouped by module with comments.

### 2.3 Translate what the apps are shown
1. **`respond()` in `api.php` and in `media.php`:**
   ```php
   if (isset($data['message']) && is_string($data['message'])) $data['message'] = tr($data['message']);
   ```
   Every `bad('literal')` and `good(['message' => 'literal'])` is then covered with no call-site change. First `grep "'message' =>"` to confirm that no response puts user-written text in `message`.
2. **Messages built from variables** (`bad("… $x …")`, `bad('…' . $x . '…')`, and the media quota message's "1 GB") become `bad(tr('… {name} …', ['name' => $x]))`. The second `tr()` inside `respond()` then finds no Spanish key for the already-Spanish text and leaves it alone. Find them with `grep -nE "bad\(\"|bad\('[^']*' *\." -r src *.php`.
3. **Fill `es.php`** with every literal message. Translate developer-only ones too ("Unknown action"); they're cheap, and skipping them only adds review noise.

### 2.4 The language setting
Next to `handle_updateTheme` in `src/Users/handlers.php`:
```php
function handle_updateLanguage($pdo, $user) {
    $lang = $_POST['lang'] ?? '';
    if (!in_array($lang, SUPPORTED_LANGS, true)) bad('Invalid language', 400);
    $pdo->prepare('UPDATE users SET lang = ? WHERE id = ?')->execute([$lang, $user['sub']]);
    respond(good(['message' => 'Language updated']));
}
```
- Register `'updateLanguage' => 'handle_updateLanguage'` in `api.php`'s handler map.
- `getMyInfo` adds `lang` to its SELECT and returns `'lang' => $row['lang']` (null when never chosen).
- `finishRegister` stores `explicitRequestLang()` in the new user's `lang`, so new accounts start in the language they signed up in.

### 2.5 Push notifications and emails
- **Push:**
  - `notificationText($actorEmail, $type, $lang)` uses `tr('{actor} liked your post', ['actor' => $actorEmail], $lang)` and so on for each type.
  - `pushNotification` reads the recipient's language once (`SELECT lang FROM users WHERE id = ?`, NULL → `'en'`) and passes it in.
  - Push titles stay "Simple Social", the brand name.
  - The service worker's fallback body in `web/public/sw.js` ("You have a new notification") only shows if a push arrives without a body, which the server never sends. Leave it.
- **One-time-code emails** in `src/Auth/handlers.php` (password reset and registration): translate the subject and body with `tr()` (request language).
  - Spanish subjects: "Tu código de Simple Social para restablecer la contraseña" and "Tu código de registro de Simple Social".
  - Keep the code on its own line, as now.
- **Moderation report emails to the admin:** English (`$lang = 'en'`).

### 2.6 API docs
Add `updateLanguage`, the `lang` field of `getMyInfo` and the `X-SS-Lang` header to the API docs (`web/src/content/api-docs.html`), in English.

### 2.7 Verify on the local bench
- `curl` a failing login with and without `-H 'X-SS-Lang: es'`: Spanish only with the header, unchanged English without it.
- `updateLanguage` accepts `es` and `en`, rejects `fr` with 400, and `getMyInfo` reflects it.
- A new account registered with the header has `lang` set; one registered without it has NULL.
- **Push:** with the mock Expo server used for the Expo push work, a follow notifies a `lang='es'` user in Spanish and a NULL user in English.
- **Email:** run the bench with `php -d sendmail_path=<a script that appends stdin to a file> -S …`. The reset email is in Spanish when requested with the header.
- **[[sstests]] check:** `sstests/backend/i18n/check-messages.php <ssapi path>` lists every literal passed to `bad(`, `tr(` or `'message' =>` that's missing from `es.php`, and every placeholder mismatch. It exits non-zero on either. Run it in the bench checks from now on.

**Commits:** `users.lang + updateLanguage`, `i18n: tr() and Spanish messages`, `i18n: push and emails in the recipient's language`, `api docs: language`.

---

## 4. Phase 3: web app (about 1.5–2 days)

### 3.1 Provider and sync
- **`web/src/lib/i18n.tsx`:** `I18nProvider` wraps **outside** `AuthProvider`, because the login pages need it. It exposes `useI18n()` → `{ lang, setLang, tr, trn }`, where `tr`/`trn` are `translate`/`translatePlural` bound to the current language.
  - The initial language follows L3: `?lang=`, then `localStorage` `ss_lang`, then `langFromTags(navigator.languages)`. Wrap storage access in try/catch.
  - Keep a module-level `currentLang` for the API client's `getLang` in `web/src/lib/api.ts`, since the client isn't a React component.
  - On every change, set `document.documentElement.lang`.
- **`<LanguageSync />`** goes inside `AuthProvider` and does L3's account sync: pending choice up, the account's choice down, or send up when NULL.
- **`core/types.ts`:** `User` gains `lang: Lang | null`, and `userFrom` maps it.
- Switching language re-renders immediately, with no reload.

### 3.2 The picker
- **Settings:** a new card right after Appearance. Its title is always bilingual, **"Language · Idioma"**, so someone stuck in the wrong language can find it. It holds a `Select` (like Theme) with "English" and "Español", saves optimistically and reverts with an error message on failure (copy `handleThemeChange`).
- **Login, Register and Landing pages:** a small "English · Español" switch at the bottom, for people who aren't logged in yet. Changing it there sets `ss_lang_pending`.

### 3.3 Convert the screens
Go group by group, one commit each, replacing every user-facing literal with `tr('…')`. That covers text, `placeholder`, `title`, `aria-label`, `alt`, toast and error messages, and empty states.
1. Layout: `Header`, `ThumbNav`, `Layout`, `SessionExpiredModal`.
2. Auth: `LoginPage`, `RegisterPage`, `ResetPasswordPage`, `OtpAuthFlow`.
3. Posts: `FeedPage`, `PostPage`, `PostCard`, `CommentItem`, likes and follow popovers, media players.
4. Create post: `CreatePostPage`, `CaptureModal`, `MediaPreview`, the media quota message.
5. Notifications (via core's `notificationText`), Profile, Search.
6. Settings, Devices, Blocked users, `ReportDialog`, `BlockUserDialog`, `TermsGate`.

Theme names become `settings.theme.*` keys: Light → Claro, Dark Blue → Azul oscuro, Dark → Oscuro, Red → Rojo, Light Blue → Azul claro, Hacker → Hacker. Also check `document.title` and anywhere else text is set outside JSX.

### 3.4 The long pages
Long text doesn't belong in a key-per-sentence dictionary. Give each translated page a Spanish twin component and pick one by language: `web/src/content/terms.en.tsx` and `terms.es.tsx` (next to the existing `api-docs.html`), with `TermsPage` rendering one or the other. The same goes for **Landing, Download, About, Conduct, Terms and Privacy**.
- **Download matters most:** it's how family members install the Android app.
- **Terms and Privacy (DECISION):** the default is a Spanish translation that opens with "Esta traducción se ofrece para tu comodidad. Si hay diferencias, prevalece la versión en inglés." Dave confirms; this isn't legal advice. A translation **doesn't change the terms version**, so nobody is asked to accept again.
- **API, Roadmap, History and Admin** stay English. In Spanish mode they get the one-line "only in English" note.

### 3.5 Stop English creeping back in
In `web/eslint.config.js`, add ESLint's built-in `no-restricted-syntax` for `web/src/**/*.tsx`:
- `JSXText[value=/[A-Za-z]{2,}/]` → "Use tr() for text people see."
- `JSXAttribute[name.name=/^(placeholder|title|alt|aria-label|label)$/] > Literal[value=/[A-Za-z]/]`.

Exclude `components/ui/**` (shadcn primitives), `content/**` and the English-only pages. Make it an error once the conversion is done.

### 3.6 Verify
- **Checks:** `pnpm typecheck`, `pnpm lint` and `pnpm test` pass.
- **Playwright smoke test** (Chromium is pre-installed):
  - Log in on a bench account, switch Settings to Español, reload: Settings is in Spanish.
  - Log out: the login page is in Spanish, and a wrong password gives a Spanish server error.
  - `/terms?lang=en` shows English.
- **Manual pass at phone width (375 px), in Spanish:** walk through every page. Spanish runs about 20–30% longer, so check buttons, the thumb-bubble labels, dialogs and the Settings rows for wrapping or cut-off text.

---

## 5. Phase 4: phone app (`ssreact/mobile/`, about 1–1.5 days)

1. **Device language:** run `npx expo install expo-localization` (SDK 57 docs first), then use `langFromTags(getLocales().map((l) => l.languageTag))`.
2. **Storage:** add `LANG_KEY` and `LANG_PENDING_KEY` to the `KEYS` list in `mobile/src/platform/tokenStore.ts`. Only keys in that list are loaded at app start.
3. **Provider:** `mobile/src/i18n/I18nProvider.tsx` + `useI18n()` → `{ lang, setLang, tr, trn }`, the same L3 rules and sync as the web. Add `getLang` in `mobile/src/platform/api.ts`. Remember L2: it's `tr`, because `t` is the theme.
4. **Picker:**
   - Settings gets a "Language · Idioma" card right after Appearance, built from the existing `Choice` component.
   - The login screen gets the same "English · Español" switch as the web.
   - Update the order in the comment at the top of `settings.tsx`.
5. **Pages the app opens in the browser** (Settings' info links and `TermsGate`): add `?lang=${lang}`, so the website matches the app.
6. **Convert every screen and component**, group by group as on the web. Use core's `notificationText` in the notifications tab. **Check text length on a small phone:** tab labels, the thumb-bubble labels (which fan out by length) and buttons.
7. **iPhone permission prompts** (camera, microphone, photos):
   - These are native strings, set today in `app.config.ts` through the `expo-camera` / `expo-image-picker` plugin options. Add Spanish versions with Expo's `locales` app config (`mobile/locales/es.json`), checking the SDK 57 "Localization" docs for the exact format.
   - This also makes the App Store list Spanish as a supported language.
   - Limitation: iOS picks these strings from the **phone's** language, not the in-app choice. That's acceptable; note it in the ssreact note.
   - Android's permission dialogs come from the system and are already in the phone's language.
   - Optional, only if SDK 57 supports it simply through `expo-localization`'s config plugin: per-app language in the phone's own settings (iOS, Android 13+). Otherwise skip it.
8. **Release note for Dave:** `expo-localization` and the `locales` config are native changes, so this needs a **new build** (APK + TestFlight). An `eas update` alone won't deliver it.
9. **Verify** (`npx expo lint`, `npx tsc --noEmit`), then on a development build:
   - On a phone set to Spanish, first launch is in Spanish.
   - Switching to English in Settings changes the whole app at once.
   - Log out, switch to Español on the login screen, log in: it stays Español, and `getMyInfo` now says `es`.
   - A push notification arrives in the chosen language.
   - On an iPhone set to Spanish, the camera permission prompt is in Spanish.

---

## 6. Phase 5: review and release (Dave + a Spanish speaker)

1. **Review the Spanish.** The agent's translation is a draft, so a native-speaking family member reviews it in two ways:
   - **In context:** using the app in Spanish (dev, plus the new phone build). This catches most problems.
   - **As a list:** `pnpm --filter @ss/core i18n:table` prints a Markdown table (key | English | Español), plus the long pages and `es.php`. Add that small script in Phase 1. Don't commit its output.

   Fixes go back as edits to `es.ts`, `es.php` and the `*.es.tsx` pages.
2. **Deploy order (Dave), dev first and then prod for each:**
   1. **ssapi.** Without the header nothing changes, so this is safe ahead of the apps. Back up the database first, as with every migration.
   2. **The web app.**
   3. **The phone build.**
3. **Store listings (optional, with phone-plan Phases 9/10):**
   - App Store Connect: a Spanish (Mexico) localization (name, subtitle, description, keywords, screenshots).
   - Google Play: a Spanish (Latin America, es-419) listing.

   The app itself works in Spanish without these.

---

## 7. Estimate and decisions

| Phase | Work | Effort |
|---|---|---|
| 1 | Core dictionaries, shared text, API header, tests | about half a day |
| 2 | Server: `users.lang`, `tr()`, Spanish messages, push and emails | half a day to a day |
| 3 | Web: provider, picker, every screen, six long pages, lint guard | 1.5–2 days |
| 4 | Phone: provider, picker, every screen, permission prompts | 1–1.5 days |
| 5 | Review fixes | about half a day, plus the reviewer's time |

About **3–5 days of agent work**. There are roughly 600–900 strings across both apps plus six long pages; most of the time is converting screens and writing good Spanish, not the plumbing. (The media plan went faster than estimated, so this may too.)

**Decisions for Dave (defaults in brackets):**
1. Spanish variety [neutral Latin American, "tú"], or Spain's Spanish.
2. Which long pages to translate [Landing, Download, About, Conduct, Terms, Privacy; API, Roadmap, History and Admin stay English].
3. Spanish Terms/Privacy with an "English version prevails" note [yes].
4. Who reviews the Spanish [a native-speaking family member].
5. A language switch on the login/register pages as well as in Settings [yes].
