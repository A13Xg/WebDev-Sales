# WebDev-Sales

## What This Is

A GitHub Pages platform for generating and demoing client websites as part of a web development sales workflow. An internal password-protected dashboard lives at the root URL and auto-maps all generated client sub-sites. When a new client is added to a JSON intake file, a local AI pipeline researches the business, designs a content-matched theme, and builds a pure HTML/CSS/JS demo site in its own subfolder — ready to send directly to the prospect.

## Core Value

A prospect receives a personalized demo URL showing their business already designed — before they've paid a cent.

## Requirements

### Validated

- ✓ Root dashboard with simple JS password protection, auto-discovers and lists all client sub-sites — Phase 01+03
- ✓ Client sub-sites served at `/[clientSlug]/` paths via GitHub Pages folder routing — Phase 02+05
- ✓ JSON intake file supports: business name, existing domain (optional), industry/business type — Phase 01+02
- ✓ Local AI pipeline reads intake, researches each business (fetches existing site if domain provided), then designs and builds the site — Phase 02+04+05
- ✓ AI design strategy matches colors, typography, and layout patterns to the business content type — not just industry label — Phase 02+05
- ✓ Generated sites draw component patterns from `../Web-Dev-Samples/docs/sites/` as style references — Phase 02+05
- ✓ Generated sites are pure HTML/CSS/JS (no build step, GitHub Pages compatible) — Phase 02+05
- ✓ Dashboard displays all client sub-sites with metadata (name, industry, slug link, status badge, copy URL) — Phase 01+05

### Active

(All v1 requirements validated — see Validated above)

### Out of Scope

- Server-side visitor logging — GitHub Pages is static; skipped entirely
- GitHub Actions CI/CD pipeline — generation runs locally via Claude Code CLI, not automated on push
- Frontend frameworks (React, Vue, etc.) — pure static files only
- E-commerce or CMS functionality in generated sites — demo/presentation only

## Context

- **Repo**: `WebDev-Sales` on GitHub, served via GitHub Pages — public repo, demos publicly accessible
- **Style reference**: `../Web-Dev-Samples/docs/sites/` contains 13 full themed examples (minimalist portfolio, gourmet restaurant, tech startup, fashion e-commerce, travel, fitness, agency, news, music festival, law firm, outdoor, indie game, theme showcase) plus a 100-theme catalog in `13-theme-showcase`
- **Generation approach**: Claude Code CLI run locally ingests `_pipeline/intake.json`, iterates each new entry, researches via web fetch (3-path fallback), picks/adapts design patterns, writes files to `/[clientSlug]/`
- **Dashboard routing**: GitHub Pages serves subdirectories naturally — no special config needed for `/clientSlug/index.html` to resolve
- **Target use**: Sent directly to prospects as `github.io/[repo]/[clientSlug]` — they see their own brand, professionally designed
- **Current state (v1.0)**: 6 live client demos, password-protected dashboard, 7-step pipeline with 3-path research fallback fully operational; per-client `buildLog.md` and `email.md` artifacts generated locally (gitignored)
- **Cloudflare note**: ~20% of small business domains are Cloudflare-proxied; Path B (web search fallback) validated and working — enrich `notes` field for best results on blocked domains

## Constraints

- **Hosting**: GitHub Pages — static files only, no server-side logic
- **Output format**: Pure HTML/CSS/JS — no build toolchain, no npm required for generated sites
- **Style source**: Must reference `../Web-Dev-Samples` component patterns; no external UI frameworks
- **Pipeline**: Local execution via Claude Code CLI — not automated CI/CD
- **Password protection**: Client-side JS only (dashboard) — not cryptographically secure, just a friction gate

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Pure HTML/CSS/JS for generated sites | No build step — GitHub Pages serves files directly, zero friction for demos | Delivered: 6 live client demos, all self-contained, served via GitHub Pages with no build toolchain |
| Local AI pipeline (not GitHub Actions) | More control, can use interactive Claude Code features and skills | Delivered: `/generate-client` skill + 7-step pipeline, 3-path research flow (A: live domain, B: Cloudflare fallback, C: notes-only) |
| JS password protection on dashboard | Static hosting constraint; purpose is friction, not real security | Delivered: SHA-256 hash comparison via WebCrypto + sessionStorage gate; no npm dependency |
| Content-type-matched design strategy | Demo effectiveness depends on client feeling it was made for them | Validated: coffee (warm/sensory/craft), plumbing (trust/authority), photography (gallery/editorial) each received distinct treatment beyond industry label |
| Web-Dev-Samples as pattern library | Existing curated themes provide high-quality starting points and maintain style consistency | Validated: all 5 generated sites reference specific theme IDs (02-gourmet-restaurant, 10-law-firm, 01-minimalist-portfolio, etc.) |
| Status enum: pending\|live\|archived\|error | Four-state pipeline state machine — error is non-blocking, retried on next run | Delivered: Phase 04 hardening + Phase 05 error badge in dashboard |
| CSS Variable Discipline for all tokens | Enables per-client theme overrides via :root without touching HTML | Delivered: enforced across index.html and all 5 generated client sites; 0 hardcoded hex in CSS selectors |

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
*Last updated: 2026-06-23 after v1.0 milestone close*
