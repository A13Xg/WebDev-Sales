# Skill: /generate-client

Generates a fully self-contained HTML/CSS/JS demo site for each pending client in `_pipeline/intake.json` and registers the result in `clients.json`.

## Invocation

```
/generate-client
```

Run this command inside a Claude Code CLI session at the repo root. CLAUDE.md is auto-loaded and provides the pipeline contract. No arguments are required — the skill reads all pending entries from `_pipeline/intake.json` automatically.

---

## Prerequisites

Before starting, verify all of the following are true:

1. `_pipeline/intake.json` exists and contains at least one entry in the `"intake"` array.
2. `clients.json` exists with a top-level `{ "clients": [...] }` wrapped structure.
3. `../Web-Dev-Samples/docs/sites/` is accessible as a sibling repo (check: does `../Web-Dev-Samples/docs/sites/01-minimalist-portfolio/` exist?).
4. `CLAUDE.md` is present at the repo root (auto-loaded by Claude Code CLI — confirms pipeline contract is active).

If any prerequisite fails, stop and report the specific missing item before proceeding.

---

## Idempotency Rule

Before processing any intake entry:

1. Read `clients.json`. Parse the `"clients"` array.
2. For each intake entry, check whether a matching `slug` already exists in `clients.json` with `"status": "live"`.
3. If a match is found with `status: "live"`: skip that entry. Log: `Skipping [businessName] — already live`.
4. If no live match is found: queue the entry for generation.

This rule is idempotent — re-running the pipeline after a successful generation will never overwrite a completed site.

---

## Pipeline Procedure (5 Steps, Executed Atomically Per Client)

Process queued entries sequentially (one at a time). Complete all 5 steps for one client before moving to the next. Do NOT process multiple clients in parallel.

---

### Step 1 — Read Intake

**Source file:** `_pipeline/intake.json`

1. Read `_pipeline/intake.json`. Parse the `"intake"` array.
2. Extract all entries.
3. Apply the idempotency rule (see above) to determine which entries to queue.
4. Log the queue: `Queued for generation: [businessName] ([slug])`.
5. If the queue is empty after idempotency check: log `All entries already live — nothing to generate.` and stop.

**Slug validation (security gate — T-02-01-02):**

Before using a slug as a directory name, validate it matches the pattern `^[a-z0-9][a-z0-9-]*[a-z0-9]$` (lowercase alphanumeric and hyphens only, no leading/trailing hyphens). Reject any slug containing `../`, spaces, `%`, `/`, or special characters. If slug is invalid, stop and log: `ERROR: Invalid slug "[slug]" — must be lowercase alphanumeric + hyphens only. Fix intake.json and re-run.`

---

### Step 2 — Research Business

**Goal:** Gather enough information about the business to write tailored, plausible copy and make an informed theme selection.

**Fallback chain (attempt in order, stop at first success):**

**Path A — Domain fetch (if `domain` field is a non-null URL):**
- Attempt to fetch and parse the business's existing website at `domain`.
- If response is 200 and parseable HTML: extract business description, services offered, target audience, brand signals (colors mentioned, imagery references, tone of voice). Log: `Research: domain — fetched [domain]`.
- If domain is null, returns non-200, is blocked by Cloudflare/WAF, or times out: fall through to Path B.

**Path B — Web search fallback:**
- Run a web search for `"[businessName] [businessType]"` (include city/region if available from notes).
- Extract: business description, services offered, audience signals. Log: `Research: web search fallback — query "[businessName] [businessType]"`.
- If search yields no usable results: fall through to Path C.

**Path C — Intake notes fallback:**
- Use the `notes` field from intake.json as the primary content signal.
- Use `businessType` and `industry` to infer standard offerings and audience.
- Log: `Research: intake notes fallback — using notes + businessType inference`.

**accentColor validation (security gate — T-02-01-01):**

If the `accentColor` field is non-null, validate it matches the pattern `^#[0-9A-Fa-f]{6}$` (7-character hex color, e.g., `#2d6a4f`). Reject data URLs, gradient strings, `rgb()` values, or any other format. If invalid: log `WARNING: accentColor "[value]" is not valid hex format — ignoring and deriving from theme.` and treat it as null.

**Research output to carry forward:**
- Business description (1–3 sentences)
- Services offered (bullet list)
- Target audience (who the site should speak to)
- Brand signals (color references, adjectives, imagery style)
- Which research path was taken (for QA documentation)

---

### Step 3 — Select Theme

**Goal:** Choose a design reference that matches the business's emotional content type.

**Rule:** Match by emotional content type, NOT industry label alone. A florist and a hardware store are both "Retail" but need different themes. A yoga studio and a gym are both "Fitness" but convey different emotional intents.

**How to select:**

1. Open `../Web-Dev-Samples/docs/sites/` and review the 13 sample sites:

   | Folder | Name | Emotional Type | Best for businessType |
   |--------|------|----------------|----------------------|
   | 01-minimalist-portfolio | Minimalist Portfolio | Peaceful, whitespace, refined | Freelancer, designer, photographer, solo consultant |
   | 02-gourmet-restaurant | Gourmet Restaurant | Luxury, high-end, sensory | Restaurant, cafe, fine dining, upscale experiences |
   | 03-tech-startup | Tech Startup | Modern, energetic, forward-looking | SaaS, software, tech agency, startup |
   | 04-fashion-ecommerce | Fashion E-commerce | Trendy, visual-first, aspirational | Retail, fashion, boutique |
   | 05-travel-adventure | Travel Adventure | Adventure, narrative-driven, wanderlust | Tour operator, travel agency, adventure sports |
   | 06-fitness-studio | Fitness Studio | Energetic, community-focused, motivating | Gym, fitness studio, yoga, wellness |
   | 07-creative-agency | Creative Agency | Bold, project-focused, confident | Design agency, creative services, media |
   | 08-news-magazine | News Magazine | Editorial, content-first, authoritative | Blog, publication, content library |
   | 09-music-festival | Music Festival | Energetic, event-focused, celebratory | Event, festival, concert, entertainment |
   | 10-law-firm | Law Firm | Trust, authority, formal, professional | Law firm, professional services, consulting, finance |
   | 11-outdoor-trail-guide | Outdoor Trail Guide | Nature-forward, trust-based, grounded | Outdoor recreation, guide service, adventure tour |
   | 12-indie-game-dev | Indie Game Dev | Playful, creative, cyberpunk-influenced | Game studio, creative software, entertainment tech |
   | 13-theme-showcase | The Theme Atlas | 100 named design themes with live samples | **Master reference — start here to explore palettes** |

2. Read the HTML/CSS of 1–2 sample sites that most closely match the emotional intent. Study: section structure, `:root` CSS variable naming, typography choices, color palette rationale.

3. Optionally read `../Web-Dev-Samples/docs/sites/13-theme-showcase/index.html` (the Theme Atlas) to explore named themes with pre-built `:root` CSS blocks. Use when you need a specific palette variation (dark/light/luxury/minimal).

4. Choose the primary reference and record the `themeRef` value (e.g., `"06-fitness-studio"` or `"13-theme-showcase:theme-name"`).

5. Document your selection: which sample folder inspired the layout/structure, which palette/vibe was chosen, and why it matches the business emotional intent.

**Reminder (D-07 — Web-Dev-Samples is inspiration only):** NEVER copy HTML blocks directly from Web-Dev-Samples samples. Study the structure, understand the pattern, then generate new markup tailored to the target business.

---

### Step 4 — Write Site

**Output file:** `[slug]/index.html`

Generate a fully self-contained pure HTML/CSS/JS file. The file must satisfy all of the following requirements:

**CSS Token Requirements (CLAUDE.md §CSS Variable Discipline):**

All color, spacing, font, and sizing values must be defined as `:root` CSS custom properties. No hardcoded hex values or bare pixel values in selectors — only `var()` references.

```css
:root {
  /* Spacing (8-point scale; 4px exception for icons only) */
  --space-xs: 4px;
  --space-sm: 8px;
  --space-md: 16px;
  --space-lg: 24px;
  --space-xl: 32px;
  --space-2xl: 48px;

  /* Typography */
  --font-family-base: Inter, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  --font-size-body: 16px;
  --font-size-label: 12px;
  --font-size-heading: 24px;
  --font-size-display: 36px;
  --font-weight-regular: 400;
  --font-weight-bold: 600;

  /* Colors — define based on theme selection + business signals */
  --color-bg: [theme-bg-color];
  --color-surface: [theme-surface-color];
  --color-accent: [accentColor if provided, else theme-accent-color];
  --color-text-primary: [theme-text-primary];
  --color-text-secondary: [theme-text-secondary];
  --color-border: [theme-border-color];
}

/* Consume tokens via var() — never hardcode values in selectors */
.card {
  background-color: var(--color-surface);
  padding: var(--space-md);
}
```

**accentColor mapping (D-04):** If `accentColor` is provided in intake and passed validation in Step 2, map it directly to `--color-accent` in `:root`. It anchors the entire palette — derive supporting colors (surface, border, hover states) from it. If `accentColor` is null, derive `--color-accent` from the selected theme.

**Asset and dependency rules:**
- All asset paths must be relative: `./css/style.css`, `./images/photo.jpg` — never `/` prefix
- Only Google Fonts and Unsplash are allowed as external CDN dependencies
- No other external CDN (no Bootstrap, no Tailwind CDN, no jQuery CDN)
- Inline all CSS in a single `<style>` block OR use a single `./css/style.css` file — not both
- Inline all JS at the bottom of `<body>` OR use a single `./js/main.js` file — not both

**Required content placements (SITE-01):**
- Business name in the `<title>` tag (e.g., `<title>Sunrise Yoga Studio</title>`)
- Business name in the primary navigation (logo text or nav heading)
- Business name in the hero section `<h1>` or as a prominent display heading

**Copy requirements (SITE-03):**
- All body copy must be tailored to this specific business — no lorem ipsum, no placeholder text
- Section content must be plausible for the businessType and industry
- Testimonials (if included) should reference the business and sound natural

**Accessibility baseline:**
- All `<img>` tags must include descriptive `alt` text
- Semantic HTML5 structure: `<nav>`, `<header>`, `<main>`, `<section>`, `<footer>`
- Color contrast: text on background must meet WCAG AA (4.5:1 for body text)

**Dynamic content safety (XSS prevention):**
- Any content inserted via JavaScript must use DOM APIs: `element.textContent = value` or `document.createElement(...)`
- Never use `innerHTML` with data derived from research, intake fields, or any external source
- Inline script content is acceptable for interactivity (mobile menu, smooth scroll, form handling)

**Mobile-first responsive layout (SITE-04):**
- Base styles: single-column layout that works at 320px viewport with no horizontal scroll
- Tablet breakpoint (768px+): multi-column layouts activate
- Desktop breakpoint (1024px+): full multi-column, max-width container centered
- All interactive elements (buttons, links) must be at least 44px × 44px tap target on mobile

**Typical section structure (AI has full discretion — SITE-05):**
- Navigation: business name + nav links (anchor-based) + primary CTA
- Hero: headline with business name + subheading + CTA button (may use Unsplash background or CSS gradient)
- Services / Offerings: what the business provides (grid or list layout)
- About / Story: business background, founder info if applicable
- Testimonials: 2–3 client quotes (plausible for the business)
- Contact: phone, email, address, and/or embedded form (form is decorative — no backend)
- Footer: business name, copyright, quick links

AI may add, remove, or adapt sections based on what makes sense for the specific businessType.

---

### Step 5 — Update Manifest

**Target file:** `clients.json`

1. Read `clients.json`. Parse the `"clients"` array.
2. Find the entry where `slug` matches the generated client's slug.
3. If no entry exists for this slug: add a new object to the `"clients"` array with all fields.
4. Update (or set) the following fields:
   - `"status": "live"`
   - `"themeRef": "[chosen theme reference — e.g., '06-fitness-studio']"` (non-null string)
   - `"generatedAt": "[current ISO8601 timestamp — e.g., '2026-06-20T21:00:00Z']"` (ISO8601 format)
5. Preserve all other existing fields on the entry (businessName, industry, businessType, domain, notes) — do not overwrite them with null or remove them.
6. Write the updated `clients.json` back to disk.
7. Verify the written file: re-read it and confirm the entry has `status: "live"`, `themeRef` is a non-null string, and `generatedAt` matches the ISO8601 pattern `^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}Z$`.

**Idempotency confirmation:** If the entry already has `status: "live"` at this point (race condition or re-run), log `Skipping manifest update — [businessName] already live.` and do not overwrite.

---

## Pre-Commit Quality Checklist

Before staging or committing any generated files, verify ALL of the following:

- [ ] `<title>` tag contains the business name (not "Untitled", "Page", or placeholder)
- [ ] Business name is visible in the primary navigation/header
- [ ] Hero section `<h1>` or display heading contains the business name
- [ ] No "lorem ipsum", "Lorem Ipsum", "placeholder", "Company Name", "Your Business", or "ACME" text anywhere on the page
- [ ] Every CSS selector uses `var()` for color, spacing, and font values — no hardcoded hex values or bare px in selectors
- [ ] All local asset paths use relative format (`./css/style.css`, not `/css/style.css`)
- [ ] No external CDN except Google Fonts and Unsplash
- [ ] Mobile layout renders correctly at 320px viewport — no horizontal scroll, readable text, tappable buttons
- [ ] All `<img>` tags have descriptive `alt` text
- [ ] No `innerHTML` assignments with data from external/intake sources — DOM APIs only
- [ ] `clients.json` updated: `status: "live"`, `themeRef` is non-null string, `generatedAt` is ISO8601 timestamp
- [ ] `_pipeline/intake.json` is NOT staged (it is gitignored — never commit it)

---

## Anti-Patterns to Avoid

**Do NOT copy HTML blocks from Web-Dev-Samples.** Read the structure for inspiration, then generate new markup for the specific business. Warning sign: generated site contains unchanged copy from a sample site.

**Do NOT hardcode hex values in selectors.** All color values must be `var(--color-*)` references. Hardcoded values (`background-color: #2d6a4f`) prevent theme adaptation in later phases.

**Do NOT use absolute paths.** `/css/style.css` breaks when GitHub Pages serves the site at `/WebDev-Sales/[slug]/`. Always use `./css/style.css`.

**Do NOT use `innerHTML` with research data.** Business name, extracted copy, or any external content inserted into the DOM must use `element.textContent = value` or DOM creation APIs. `innerHTML` with untrusted data is an XSS vector.

**Do NOT commit `_pipeline/intake.json`.** It is gitignored and contains operator-local business data. Only commit the generated `[slug]/index.html` and the updated `clients.json`.

**Do NOT skip the idempotency check.** Always verify `status: "live"` in `clients.json` before generating. Regenerating a live site overwrites recent operator changes.

---

## File Path Reference

| File | Role in Pipeline |
|------|-----------------|
| `_pipeline/intake.json` | Step 1: operator input — read only, never written by pipeline |
| `clients.json` | Step 1 (idempotency check) + Step 5 (manifest update) |
| `../Web-Dev-Samples/docs/sites/` | Step 3: theme inspiration library (13 sample sites + Theme Atlas) |
| `[slug]/index.html` | Step 4: generated site output — self-contained HTML/CSS/JS |
| `CLAUDE.md` | Always active: pipeline contract, CSS Variable Discipline, GitHub Pages rules |
