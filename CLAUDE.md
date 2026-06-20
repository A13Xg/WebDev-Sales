<!-- GSD:project-start source:PROJECT.md -->

## Project

**WebDev-Sales**

A GitHub Pages platform for generating and demoing client websites as part of a web development sales workflow. An internal password-protected dashboard lives at the root URL and auto-maps all generated client sub-sites. When a new client is added to a JSON intake file, a local AI pipeline researches the business, designs a content-matched theme, and builds a pure HTML/CSS/JS demo site in its own subfolder — ready to send directly to the prospect.

**Core Value:** A prospect receives a personalized demo URL showing their business already designed — before they've paid a cent.

### Constraints

- **Hosting**: GitHub Pages — static files only, no server-side logic
- **Output format**: Pure HTML/CSS/JS — no build toolchain, no npm required for generated sites
- **Style source**: Must reference `../Web-Dev-Samples` component patterns; no external UI frameworks
- **Pipeline**: Local execution via Claude Code CLI — not automated CI/CD
- **Password protection**: Client-side JS only (dashboard) — not cryptographically secure, just a friction gate

<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->

## Technology Stack

## Recommended Stack

### Root Dashboard (`/index.html`)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Vanilla HTML/CSS/JS | n/a | Dashboard shell | No build step; GitHub Pages serves directly; zero toolchain friction |
| `sessionStorage` API | native | Auth state persistence | Tab-scoped — clears on close; appropriate for internal friction gate |
| `crypto.subtle` (WebCrypto) | native | Password hash comparison | Browser-native, no dependency; HTTPS required (GitHub Pages always HTTPS) |
| `fetch()` + `clients.json` | native | Client manifest loading | Standard pattern — dashboard fetches the manifest after auth, renders cards |

### Client Sub-sites (`/[clientSlug]/`)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Pure HTML/CSS/JS | n/a | Demo sites | No build step; GitHub Pages serves `index.html` from each subfolder natively |
| Custom CSS (per site) | n/a | Themed styling | Inline or single `style.css` per client; no shared framework that could drift |
| No external CDN dependencies | — | Portability | Demo survives CDN outages; fully self-contained in the repo |

### AI Generation Pipeline (local, not deployed)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Claude Code CLI | latest | Site generation orchestrator | Already in use; reads `clients.json`, writes files, fetches web context |
| `CLAUDE.md` | n/a | Project context injection | Auto-loaded every session; encodes intake schema, output rules, path conventions |
| Custom slash command / skill | `.claude/skills/` | `/generate-client` workflow trigger | Encapsulates the full generation pipeline as a repeatable command |
| `clients.json` | n/a | Intake manifest | Single source of truth; Claude reads it, generates missing slugs, updates it |
| `../Web-Dev-Samples/docs/sites/` | n/a | Style reference library | 13 themed sites + 100-theme catalog; Claude reads patterns before generating |

## GitHub Pages Routing

### How it works

- `index.html` at repo root → `https://username.github.io/repo/`
- `clientSlug/index.html` → `https://username.github.io/repo/clientSlug/`
- `/clientSlug/` (trailing slash) → serves `clientSlug/index.html` automatically
- `/clientSlug` (no trailing slash) → GitHub Pages issues a redirect to `/clientSlug/`

### Critical: add `.nojekyll`

- Ignore files/folders prefixed with `_` (e.g. `_assets/`)
- Interfere with `node_modules/` directories (404s instead of files)
- Cause unpredictable behavior with multiple `index.html` files

### What GitHub Pages cannot do

- No server-side redirects — workaround is `<meta http-equiv="refresh">` or JS redirect in a 200-status HTML file
- No dynamic route resolution — every path must correspond to a real file or folder
- No SPA client-side routing — `/clientSlug/some-deep-path` will 404 unless that exact file exists
- No `index.json` acting as a directory listing — must fetch `clients.json` explicitly by path

### Gotchas

| Gotcha | Detail |
|--------|--------|
| `.html` extension stripping | GitHub Pages will serve `foo.html` at `/foo` — avoid naming collisions between files and folders |
| No wildcard 404 fallback for sub-paths | A `404.html` at root handles all unmatched requests; use this for a graceful "client not found" page |
| Folder redirect behavior | `/clientSlug` → redirect → `/clientSlug/` adds a round trip; use trailing slash in all internal links |
| Case sensitivity | GitHub Pages is case-sensitive (Linux server); `ClientSlug/` and `clientslug/` are different paths |

## Password Protection Approach

### The constraint

### Recommended approach: SHA-256 hash comparison in vanilla JS

### Why not StatiCrypt

- Adding an npm dependency to the workflow
- Encrypting/re-encrypting the file on every dashboard update
- The added cognitive overhead of a build step for a purely internal tool

### Why not hash-based directory redirection

- Requires creating a secret folder at the hashed path — breaks the clean folder-per-client structure
- Harder to update the password (rename the folder)
- Not suited to a dashboard that needs to render dynamic content post-auth

### Tradeoffs summary

| Approach | Security | Complexity | Suitable Here |
|----------|----------|------------|--------------|
| Plaintext comparison | Very low | Zero | No |
| SHA-256 hash + sessionStorage | Low (source-inspectable) | Minimal | **Yes** |
| StatiCrypt (AES-256 + PBKDF2) | High | Moderate (npm step) | If content is sensitive |
| Hash-based directory redirect | Low | Medium | No (wrong structure) |
| Server-side auth | Real | High (needs server) | Out of scope |

## Client Manifest Schema

### Recommended `clients.json` structure

### Field definitions

| Field | Type | Required | Purpose |
|-------|------|----------|---------|
| `slug` | string | Yes | Folder name and URL path segment — lowercase, hyphenated, URL-safe |
| `name` | string | Yes | Display name for dashboard card |
| `industry` | string | Yes | Broad category (Home Services, Retail, Food & Beverage, etc.) |
| `businessType` | string | Yes | Specific type — this drives AI design matching, not just `industry` |
| `domain` | string\|null | No | Existing domain to fetch for content research; null if none |
| `status` | enum | Yes | `pending` \| `generated` \| `error` — pipeline state machine |
| `generatedAt` | ISO8601\|null | No | Timestamp of last generation; null if pending |
| `primaryColor` | hex\|null | No | AI-chosen primary color; populated after generation, used in dashboard card |
| `themeRef` | string\|null | No | Which Web-Dev-Samples reference was used (e.g. `"02-restaurant"`) |
| `notes` | string | No | Free text for the operator; not consumed by pipeline |

### Design rationale

- **`businessType` separate from `industry`**: The PROJECT.md requirement says "design strategy matches colors, typography, and layout patterns to the business *content type*, not just industry label." A `businessType: "florist"` gets different treatment than `businessType: "hardware store"` even though both are `industry: "Retail"`. Keep both fields.
- **`status` field**: Allows the pipeline to skip already-generated clients and lets the dashboard show a "pending" badge for slots not yet built.
- **`themeRef` back-reference**: Storing which Web-Dev-Samples theme was used enables re-generation with a different theme without having to re-infer from scratch.
- **`generated` top-level timestamp**: Lets the dashboard display a "last updated" line without reading individual files.

### Dashboard consumption pattern

## AI Pipeline Stack

### How Claude Code CLI integrates

### CLAUDE.md is load-bearing

- The `clients.json` schema (fields, types, allowed status values)
- Output path conventions (`/[slug]/index.html`, no subdirectories within client folders unless needed)
- Style reference path (`../Web-Dev-Samples/docs/sites/`)
- Constraint reminders (pure HTML/CSS/JS output, no `<script src="...cdn...">` for generated sites, no build step)
- Design strategy notes (match `businessType`, not just `industry`; reference the closest Web-Dev-Samples theme before inventing)

### Web-Dev-Samples library — how to structure for AI consumption

# Web-Dev-Samples Site Index

| Folder | Name | Best for businessType | Key palette | Layout style |
|--------|------|----------------------|-------------|-------------|
| 01-minimalist-portfolio | Minimalist Portfolio | freelancer, photographer, designer | monochrome + accent | Single-column, lots of whitespace |
| 02-gourmet-restaurant | Gourmet Restaurant | restaurant, cafe, fine dining | deep red, gold, cream | Full-width hero, menu grid |
| 03-tech-startup | Tech Startup | SaaS, software, tech agency | navy, electric blue | Features grid, CTA sections |
| ...etc |

### Slash command / skill structure

### Pipeline constraints

- Generation is sequential per client (one at a time), not parallel — avoids context window confusion between clients
- The pipeline updates `clients.json` atomically per client (update status before moving to next)
- No npm install, no build step in the output — generated sites must be directly committable

## What NOT to Use

| Technology | Why Not |
|------------|---------|
| React / Vue / Svelte | Build step required; output is not plain HTML/CSS/JS; GitHub Pages needs a build pipeline or pre-built dist — adds friction and violates the project constraint |
| Jekyll | Default GitHub Pages processor creates conflicts with multi-site subfolder structure; add `.nojekyll` and skip it entirely |
| Netlify / Vercel identity auth | Requires migrating hosting away from GitHub Pages; out of scope |
| GitHub Actions for generation | Project explicitly chose local Claude Code CLI for interactivity and control; Actions would require storing API keys in repo secrets and loses the interactive Claude Code feature set |
| StatiCrypt for dashboard | Overkill for the stated purpose (friction gate, not security); adds an npm dependency and a re-encryption step on every dashboard edit |
| Tailwind CSS (CDN) for generated sites | CDN dependency in client demos is a reliability risk; generated sites must be fully self-contained |
| Shared CSS framework across all client sites | One bad CDN load breaks every client demo simultaneously; each site gets its own inline/local styles |
| `localStorage` for auth persistence | Survives browser restart — makes the "logout" model confusing; `sessionStorage` is the right scope |
| Directory listing via server (Apache `Options +Indexes`) | GitHub Pages does not support server configuration; use `clients.json` manifest instead |

## Confidence Levels

| Recommendation | Confidence | Basis |
|----------------|------------|-------|
| `.nojekyll` required | HIGH | GitHub Pages official docs + Simon Willison's authoritative TIL |
| Subfolder routing works natively | HIGH | GitHub Pages official docs; `/slug/` → `slug/index.html` is documented behavior |
| `fetch('clients.json')` for auto-discovery | HIGH | Standard static site pattern; no server required; fetch works on same-origin GitHub Pages |
| SHA-256 + sessionStorage for auth gate | HIGH | WebCrypto native in all modern browsers; HTTPS guaranteed on GitHub Pages |
| Skip StatiCrypt for dashboard | HIGH | Correct for the stated security posture (friction, not encryption) |
| `clients.json` schema design | HIGH | Schema derived directly from PROJECT.md requirements; no external dependencies |
| CLAUDE.md as pipeline context carrier | HIGH | Documented Claude Code behavior; auto-loaded every session |
| `INDEX.md` in Web-Dev-Samples for AI lookup | MEDIUM | Sound engineering practice; not tested against this specific library yet — validate in Phase 1 |
| SKILL.md for `/generate-client` command | MEDIUM | Claude Code skills are well-documented; specific pipeline encoding needs validation in practice |
| `themeRef` back-reference in manifest | MEDIUM | Design decision, not a technical constraint; may evolve once generation is running |

## Sources

- [GitHub Pages: The Missing Manual — Simon Willison](https://til.simonwillison.net/github/github-pages)
- [GitHub Pages subfolder routing discussion](https://github.com/orgs/community/discussions/58276)
- [StatiCrypt — robinmoisson/staticrypt](https://github.com/robinmoisson/staticrypt)
- [StatiCrypt README](https://github.com/robinmoisson/staticrypt/blob/main/README.md)
- [Password protection for GitHub Pages — agalera.eu](https://www.agalera.eu/github-pages-password/)
- [Client-side password verification approaches](https://yousefamar.com/memo/articles/dev/client-side-password-verification/)
- [Claude Code CLI reference — introl.com](https://introl.com/blog/claude-code-cli-comprehensive-guide-2025)
- [Using CLAUDE.md files — Anthropic blog](https://claude.com/blog/using-claude-md-files)
- [About GitHub Pages and Jekyll — GitHub Docs](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/about-github-pages-and-jekyll)

<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->

## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->

## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->

## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->

## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:

- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->

## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->

## Pipeline Sequence

The AI generation pipeline executes in 5 steps per client entry, in order. This sequence is encoded in Phase 2's `/generate-client` skill and must be followed atomically per client to maintain `clients.json` consistency.

### Step 1: Read Intake Staging File

- Source: `_pipeline/intake.json` (local, not in git)
- Action: For each entry WITHOUT a matching `status: "live"` entry in `clients.json`, queue for generation
- Idempotency: Skip entries already `status: "live"`

### Step 2: Research the Business

- Attempt to fetch and parse the business's existing `domain` (if provided)
- Fall back to web search, business directory listings, or operator `notes` field if domain is blocked or absent
- Extract: business description, services, target audience, visual and brand clues

### Step 3: Select Web-Dev-Samples Theme

- Match emotional content type, NOT industry label alone (e.g., florist vs. hardware store are both "Retail" but get different themes)
- Read `../Web-Dev-Samples/docs/sites/` and select the closest thematic reference before generating
- Rationale: Theme matching drives color, typography, and layout — must precede generation

### Step 4: Write Complete Site

- Output: `[slug]/index.html` — single pure HTML/CSS/JS file
- All asset paths are relative (no `/` prefix, no CDN except Google Fonts and Unsplash in generated sites)
- No build step, no npm, no shared assets between sites — fully self-contained
- Inline all CSS and JS, or use a single local `style.css` per site

### Step 5: Update Manifest

- Set `status: "live"` for this entry in `clients.json`
- Populate `themeRef`: which Web-Dev-Samples theme was used (e.g., `"02-restaurant"`)
- Populate `generatedAt`: ISO8601 timestamp of generation
- **Idempotency rule:** If entry already `status: "live"`, skip to next (prevent overwriting recent changes)

### Atomic Semantics

- Pipeline processes ONE client at a time (sequential, not parallel) — avoids context window confusion between clients
- Update `clients.json` after site is fully written — never leave manifest in inconsistent state
- If generation fails for one client, operator can retry independently; other clients unaffected

## CSS Variable Discipline

All styling tokens in the dashboard and generated client sites MUST be defined as CSS custom properties (`:root` variables). This enables Phase 2+ to override theme values globally without touching HTML structure.

### Required Pattern

```
:root {
  /* Spacing (8-point scale, 4px exceptions for icons only) */
  --space-xs: 4px;
  --space-sm: 8px;
  --space-md: 16px;
  --space-lg: 24px;
  --space-xl: 32px;
  --space-2xl: 48px;

  /* Typography */
  --font-family-base: Inter, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  --font-size-body: 14px;
  --font-size-label: 12px;
  --font-size-heading: 16px;
  --font-size-display: 24px;
  --font-weight-regular: 400;
  --font-weight-bold: 600;

  /* Colors (dark theme defaults) */
  --color-bg: #0f1419;
  --color-surface: #1a1f2e;
  --color-accent: #3b82f6;
  --color-text-primary: #e5e7eb;
  --color-text-secondary: #9ca3af;
  --color-border: #374151;
  --color-status-pending: #f59e0b;
  --color-status-live: #10b981;
  --color-status-archived: #6b7280;
}

/* Usage in selectors — consume via var() */
.my-element {
  background-color: var(--color-bg);
  padding: var(--space-md);
  font-size: var(--font-size-body);
}
```

### Never Do This

```
/* WRONG: Hard-coded hex values in selectors */
.my-element {
  background-color: #0f1419;  /* Should be var(--color-bg) */
  padding: 16px;              /* Should be var(--space-md) */
}

/* WRONG: Inline style attribute with hard-coded values */
/* <div style="color: #e5e7eb; padding: 8px;"> */
```

### Why This Matters

- Generated sites can override `:root` in their own stylesheet to apply per-client branding without touching HTML structure
- Dashboard remains pristine; generated sites inherit base tokens and override as needed
- Enables theme management (Phase 2+) without duplicating HTML across all sites
- Refactoring tokens (e.g., "change all spacing by 2px") is a single `:root` update, not a find-and-replace across all files

### Enforcement Checklist

- All color values are `var(--color-*)` in CSS, never hardcoded hex
- All spacing and sizing values are `var(--space-*)`, never hardcoded px
- All font properties reference `var(--font-*)` tokens
- No inline `style=""` attributes with hard-coded values
- `:root` is the canonical token definition; selectors consume via `var()`
