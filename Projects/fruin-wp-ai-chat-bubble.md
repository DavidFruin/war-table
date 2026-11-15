---
status: active
repo: https://github.com/DavidFruin/fruin-wp-ai-chat-bubble
---

# fruin-wp-ai-chat-bubble

## Summary
A WordPress plugin that adds an AI chat bubble to a site. First target is Dave's own site, fruininc.com. It does two jobs: a **sales assistant** (hosting plans, domains, website packages, web development, translation services) and a **tech support bot** that points visitors to the right documentation page.

## Status (2026-11-14)
- Repo created (private). Technical plan approved 2026-11-14; Phase 1 under way.
- What fruininc.com looks like, for the plan:
  - WordPress 7.1.4 + WooCommerce 11.3.0, with Everest Forms and a booking plugin (appointment and contact pages already exist).
  - About 213 pages, 48 WooCommerce products, 4 blog posts.
  - Bilingual. Many pages exist in both English and Spanish (e.g. Website Translation / Traducción de Sitio Web, Web Hosting / Alojamiento Web).
  - Docs are ordinary WordPress pages under `/help/` (English) and `/ayuda/` (Spanish). Mostly Church Admin guides. There's also a knowledge base page (Base de Conocimientos) and a ticket page (Enviar un Ticket).
  - Lots of junk pages the bot must not use: theme demos (Theme 1–8, Cafe Theme 1–6), booking previews, test pages, a Stripe checkout test.

## Decisions
- First site is fruininc.com — it's Dave's own business site, and the place he wants it.
- Sales + tech support in one bot — the site sells services and also hosts the product docs.
- English and Spanish from the start — the site and its docs are already bilingual. The bot replies in the visitor's language and links the doc page in that language when one exists.
- **No learning.** The bot knows only what Dave writes: an instructions box plus knowledge entries (title, text, English link, Spanish link). It never reads or indexes the site — Dave wants full control of what it says, and the site has many junk pages.
- **Model agnostic, with the plugin's own switcher** (Anthropic, or any OpenAI-style service by base URL + model name). Chosen over WordPress 7's built-in AI client so it works on any WordPress version and can have spend limits.
- **Logged-in users only.** A "Visible to" setting starts at admins only, because…
- **Testing happens on fruininc.com itself** — Dave doesn't want a separate test site.
- **Opt out of AI:** a "Talk to a person" button, plus a setting to turn that whole option on or off per site.
- **Dave answers from Telegram** (one topic per conversation in a private group); the customer stays in the bubble. A generic signed webhook API covers other messaging apps.
- **Built to also run on hosting customers' sites** — nothing Fruin-specific in code; each site has its own keys and bot. Licensing undecided.
- Plain PHP + plain JavaScript, no AI SDKs, no build step — fewer moving parts on shared hosting.

## Phases
1. AI chat (settings, knowledge entries, providers, bubble) — **in progress 2026-11-14**
2. Human handoff (on/off setting, button, email notices, wp-admin inbox)
3. Telegram connector + generic webhook API
4. Ready for hosting customers (branding, Spanish UI file, retention, self-updates)

Full plan: `docs/PLAN.md` in the repo. Dave's rule: ask before starting each phase.

## Open questions
- Licensing/pricing for hosting customers, and who pays for their AI usage.

