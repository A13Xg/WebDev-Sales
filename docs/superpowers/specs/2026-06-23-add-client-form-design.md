# Add Client Form — Design Spec

**Date:** 2026-06-23
**Status:** Approved

## Problem

Adding a new client currently requires manually editing `_pipeline/intake.json` — a JSON file that's gitignored and lives only on the operator's machine. This is error-prone and requires knowing the schema.

## Solution

Add an "Add Client" modal form to the dashboard that:
1. Captures all intake fields through a structured UI
2. Generates a valid intake JSON entry
3. Lets the operator copy, download, or write the entry directly to `_pipeline/intake.json` on disk

The `/generate-client` pipeline run remains a manual CLI step.

---

## Architecture

All changes are confined to `index.html` — no new files, no server, no build step. Vanilla JS throughout.

**New capabilities added:**
- Modal overlay with focus trap
- Form with live slug generation
- File System Access API for "Send to Pipeline" (Chromium only)
- Blob URL download for `intake.json`
- Clipboard copy of JSON entry

---

## Form Fields

### Always Visible

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| Business Name | text input | Yes | Drives slug auto-generation |
| Slug | text (derived) | Yes | Auto-generated; editable via "Edit" toggle |
| Industry | text input | Yes | e.g., "Home Services", "Food & Beverage" |
| Business Type | text input | Yes | e.g., "coffee roaster", "plumber" |
| Domain | url input | No | Placeholder: `https://example.com — optional` |
| Notes | textarea | No | Placeholder: `Research direction, services, tone, colors — optional` |

### Advanced (collapsed by default, toggled via "▸ Advanced options")

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| Services | textarea | No | Placeholder: `Drain clearing, pipe repair, emergency calls — optional` |
| Tone | text input | No | Placeholder: `trust, authoritative, warm — optional` |
| Color Hint | text input | No | Placeholder: `#1a3c5e or "navy and white" — optional` |

---

## Slug Auto-Generation

- Updates live as user types Business Name
- Transform: lowercase, spaces → hyphens, strip non-alphanumeric except hyphens
- Example: `"Artisan Coffee Roasters"` → `artisan-coffee-roasters`
- "Edit" link beneath slug field unlocks it for manual override
- Once manually edited, auto-generation pauses for that session (override is preserved)

---

## Validation

- Required fields (Business Name, Industry, Business Type): inline error message on submit if blank
- Domain: validates as URL format when non-empty (`/^https?:\/\/.+/`)
- Slug: must match `^[a-z0-9][a-z0-9-]*[a-z0-9]$` (auto-generated slug always passes; shown as error only if manually broken)
- No slug collision check — pipeline handles idempotency

---

## Output Panel

Shown after form submit — replaces the form fields inside the modal (the modal stays open; the form is hidden and the result panel is shown in its place). Contains:

1. **Generated JSON block** — `<pre><code>` rendering the full entry object:

```json
{
  "businessName": "Artisan Coffee Roasters",
  "slug": "artisan-coffee-roasters",
  "industry": "Food & Beverage",
  "businessType": "coffee roaster",
  "domain": "https://www.onyx.coffee",
  "notes": "...",
  "services": "...",
  "tone": "...",
  "colorHint": "..."
}
```

Only fields with non-empty values are included in output (omit blank optionals).

2. **Three action buttons:**

| Button | Mechanism | Behavior |
|--------|-----------|----------|
| Copy JSON | `navigator.clipboard.writeText()` | Copies the entry object; button shows "Copied!" for 2s |
| Download intake.json | Blob URL + `<a download="intake.json">` | Downloads `{ "intake": [entry] }` as `intake.json` |
| Send to Pipeline | File System Access API (`showSaveFilePicker`) | Shows a yellow inline warning "This overwrites the current intake file" next to the button. On click, opens a save file picker defaulting to filename `intake.json`; user confirms path; file is written with `{ "intake": [entry] }`. Button is hidden on unsupported browsers (replaced by a note: "Use Download on this browser"). |

3. **"Add Another" link** — resets form to blank, returns to form view.

---

## Modal Behavior

**Trigger:** "Add Client" button in dashboard header, right-aligned. Visible only post-auth. Styled as solid accent button matching existing `.gate-submit` / `.copy-btn` pattern.

**Open:**
- Dark overlay (`rgba(0,0,0,0.7)`) full-screen
- Centered card (max-width: 480px) matching `.gate-card` visual style
- First required field receives focus

**Close:**
- ✕ button in top-right corner of card
- Click outside card (on overlay)
- Escape key

**Focus trap:** Tab key cycles through focusable elements within modal only while open.

**State on close:** Form resets to blank.

---

## Styling

All new CSS tokens added to `:root` following CSS Variable Discipline. No hardcoded hex in selectors.

New tokens needed:
```css
:root {
  --color-overlay: rgba(0, 0, 0, 0.7);
  --color-code-bg: #0d1117;        /* JSON preview block background */
  --color-input-bg: var(--color-bg); /* reuse existing */
  --modal-max-width: 480px;
}
```

All other visual properties (`--color-surface`, `--color-border`, `--color-accent`, `--font-*`, `--space-*`) reused from existing `:root`. No new color values except `--color-code-bg`.

Advanced section uses `<details>`/`<summary>` HTML elements for native collapse behavior — no JS required for the toggle.

---

## Browser Compatibility

| Feature | Support |
|---------|---------|
| Modal / Clipboard / Blob download | All modern browsers |
| File System Access API (`showSaveFilePicker`) | Chromium only (Chrome 86+, Edge 86+). Not supported in Firefox/Safari. |
| "Send to Pipeline" button | Hidden via `'showSaveFilePicker' in window` feature check; replaced by a note pointing to the Download option |

---

## Files Changed

- `index.html` — only file modified. All additions are:
  - New CSS tokens in `:root`
  - New CSS classes for modal, form, output panel
  - "Add Client" button in `<header>`
  - Modal HTML structure (hidden by default)
  - JS functions: `openAddClientModal()`, `closeAddClientModal()`, `generateSlug()`, `buildIntakeEntry()`, `showIntakeResult()`, `downloadIntake()`, `sendToPipeline()`
  - Event wiring in `initDashboard()` after auth passes

---

## Out of Scope

- Appending to existing `_pipeline/intake.json` entries (Send to Pipeline overwrites)
- Triggering `/generate-client` from the browser
- Slug collision detection
- Editing or deleting existing clients from the dashboard
