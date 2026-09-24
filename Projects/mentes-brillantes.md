---
status: active
repo:
---

# mentes-brillantes

## Summary
Static mockup website (Spanish) for a K-12 robotics education business — "Mentes Brillantes" (Brilliant Minds). Pure HTML/CSS/JS, no backend. Built to show clients, not a working product yet. Not a git repo — lives at `~/dev/mentes-brillantes`, unversioned.

## Status
- **Mockup only.** Forms don't submit, no payment processing, no auth, no database — everything is visual.
- Pages: home, courses (3-level curriculum), virtual classroom (simulated Zoom-style UI), teacher profiles (5 teachers), checkout/cart (2 pre-loaded courses, $5,700 MXN total), payment methods (card/PayPal/bank transfer), login, homework submission (Moodle-style, student view).
- Tech: no framework, CSS variables for theming (`#0077B6` blue / `#2A9D8F` teal / `#90E0EF` light blue / `#CAF0F8` background), mobile-first responsive. `app.js` currently only handles the mobile menu toggle.
- Run locally: `python3 -m http.server 8000` from the project folder.

## Decisions
- No decisions log yet — project notes so far are descriptive (what exists), not why-choices. First real architectural decisions will land once it moves past mockup stage.

## Next steps
- [ ] Add a backend for form handling
- [ ] Real payment processing
- [ ] User authentication
- [ ] Student dashboard
- [ ] Video streaming for the virtual classroom
- [ ] Turn into an actual git repo once it's more than a mockup

## Links
-
