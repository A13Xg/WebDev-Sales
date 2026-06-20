# WebDev-Sales

## What This Is

A GitHub Pages platform for generating and demoing client websites as part of a web development sales workflow. An internal password-protected dashboard lives at the root URL and auto-maps all generated client sub-sites. When a new client is added to a JSON intake file, a local AI pipeline researches the business, designs a content-matched theme, and builds a pure HTML/CSS/JS demo site in its own subfolder — ready to send directly to the prospect.

## Core Value

A prospect receives a personalized demo URL showing their business already designed — before they've paid a cent.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Root dashboard with simple JS password protection, auto-discovers and lists all client sub-sites
- [ ] Client sub-sites served at `/[clientSlug]/` paths via GitHub Pages folder routing
- [ ] JSON intake file supports: business name, existing domain (optional), industry/business type
- [ ] Local AI pipeline reads intake, researches each business (fetches existing site if domain provided), then designs and builds the site
- [ ] AI design strategy matches colors, typography, and layout patterns to the business content type — not just industry label
- [ ] Generated sites draw component patterns from `../Web-Dev-Samples/docs/sites/` as style references
- [ ] Generated sites are pure HTML/CSS/JS (no build step, GitHub Pages compatible)
- [ ] Dashboard displays all client sub-sites with metadata (name, industry, slug link)

### Out of Scope

- Server-side visitor logging — GitHub Pages is static; skipped entirely
- GitHub Actions CI/CD pipeline — generation runs locally via Claude Code CLI, not automated on push
- Frontend frameworks (React, Vue, etc.) — pure static files only
- E-commerce or CMS functionality in generated sites — demo/presentation only

## Context

- **Repo**: `WebDev-Sales` on GitHub, served via GitHub Pages
- **Style reference**: `../Web-Dev-Samples/docs/sites/` contains 13 full themed examples (minimalist portfolio, gourmet restaurant, tech startup, fashion e-commerce, travel, fitness, agency, news, music festival, law firm, outdoor, indie game, theme showcase) plus a 100-theme catalog in `13-theme-showcase`
- **Generation approach**: Claude Code CLI run locally ingests `clients.json`, iterates each new entry, researches via web fetch, picks/adapts design patterns, writes files to `/[clientSlug]/`
- **Dashboard routing**: GitHub Pages serves subdirectories naturally — no special config needed for `/clientSlug/index.html` to resolve
- **Target use**: Sent directly to prospects as `github.io/[repo]/[clientSlug]` — they see their own brand, professionally designed

## Constraints

- **Hosting**: GitHub Pages — static files only, no server-side logic
- **Output format**: Pure HTML/CSS/JS — no build toolchain, no npm required for generated sites
- **Style source**: Must reference `../Web-Dev-Samples` component patterns; no external UI frameworks
- **Pipeline**: Local execution via Claude Code CLI — not automated CI/CD
- **Password protection**: Client-side JS only (dashboard) — not cryptographically secure, just a friction gate

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Pure HTML/CSS/JS for generated sites | No build step — GitHub Pages serves files directly, zero friction for demos | — Pending |
| Local AI pipeline (not GitHub Actions) | More control, can use interactive Claude Code features and skills | — Pending |
| JS password protection on dashboard | Static hosting constraint; purpose is friction, not real security | — Pending |
| Content-type-matched design strategy | Demo effectiveness depends on client feeling it was made for them | — Pending |
| Web-Dev-Samples as pattern library | Existing curated themes provide high-quality starting points and maintain style consistency | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-06-19 after initialization*
