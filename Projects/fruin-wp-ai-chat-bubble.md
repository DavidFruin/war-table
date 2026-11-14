---
status: active
repo: https://github.com/DavidFruin/fruin-wp-ai-chat-bubble
---

# fruin-wp-ai-chat-bubble

## Summary
A WordPress plugin that adds an AI chat bubble to a site. First target is Dave's own site, fruininc.com. It does two jobs: a **sales assistant** (hosting plans, domains, website packages, web development, translation services) and a **tech support bot** that points visitors to the right documentation page.

## Status (2026-11-14)
- Repo created (private, README only). Planning stage, no code yet.
- What fruininc.com looks like, for the plan:
  - WordPress 7.1.4 + WooCommerce 11.3.0, with Everest Forms and a booking plugin (appointment and contact pages already exist).
  - About 213 pages, 48 WooCommerce products, 4 blog posts.
  - Bilingual. Many pages exist in both English and Spanish (e.g. Website Translation / Traducción de Sitio Web, Web Hosting / Alojamiento Web).
  - Docs are ordinary WordPress pages under `/help/` (English) and `/ayuda/` (Spanish). Mostly Church Admin guides. There's also a knowledge base page (Base de Conocimientos) and a ticket page (Enviar un Ticket).
  - Lots of junk pages the bot must not use: theme demos (Theme 1–8, Cafe Theme 1–6), booking previews, test pages, a Stripe checkout test.

## Decisions
- First site is fruininc.com — it's Dave's own business site, and the place he wants it.
- Sales + tech support in one bot — the site sells services and also hosts the product docs.
- English and Spanish from the start — the site and its docs are already bilingual. The bot should reply in the visitor's language and link to the doc page in that language when one exists.

## Open questions
- Only fruininc.com, or also offered to Fruin Hosting customers? (Changes who pays for AI usage and how much per-site setup is needed.)
- Handoff when someone wants to buy or talk to a person: booking plugin, Everest Forms email capture, the ticket page, or just a link?
- Which pages the bot learns from: everything minus an exclude list, or only chosen sections (`/help/`, `/ayuda/`, products, service pages)?
- Defaults proposed, not yet agreed: Claude only; Dave's own API key in a settings page; corner bubble with adjustable colors and greeting; conversations kept 30 days in the WordPress database.

## Next steps
- [ ] Get answers to the open questions
- [ ] Write the plan (phases) into the repo
