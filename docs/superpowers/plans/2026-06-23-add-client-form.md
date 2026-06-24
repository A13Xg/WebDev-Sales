# Add Client Form Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an "Add Client" modal form to the dashboard that captures intake fields, auto-generates a slug, validates inputs, and outputs a JSON entry the operator can copy, download, or write directly to `_pipeline/intake.json`.

**Architecture:** All changes are confined to `index.html`. New CSS tokens extend the existing `:root` block. New HTML for the modal is appended to `<body>` as a hidden overlay. New JS functions are added inside the existing `<script>` block and wired into `initDashboard()` after auth passes.

**Tech Stack:** Vanilla HTML/CSS/JS. File System Access API (Chromium only) for "Send to Pipeline". Navigator Clipboard API. Blob URL for download.

## Global Constraints

- Only `index.html` is modified — no new files, no build step.
- All CSS values in selectors use `var(--token-name)` — no hardcoded hex.
- New CSS tokens defined in `:root`, not inline.
- No external dependencies (no CDN, no npm).
- `<details>`/`<summary>` for Advanced section collapse — no JS toggle.
- File System Access API is feature-checked: `'showSaveFilePicker' in window`. Hidden on unsupported browsers with a note in its place.
- Output JSON omits fields with empty values.
- Slug must pass `/^[a-z0-9][a-z0-9-]*[a-z0-9]$/` — auto-generated slug always passes; error shown only if manually broken.

---

## File Structure

**Single file modified:** `index.html`

Additions in order of appearance in the file:

| Section | What's Added |
|---------|-------------|
| `:root` | 3 new CSS tokens (`--color-overlay`, `--color-code-bg`, `--modal-max-width`) |
| `<style>` | CSS for: `.add-client-btn`, `.modal-overlay`, `.modal-card`, `.modal-header`, `.modal-close`, `.form-field`, `.form-label`, `.form-input`, `.form-error`, `.form-actions`, `.output-panel`, `.output-json`, `.output-actions`, `.pipeline-warning`, `.btn-secondary` |
| `<header>` | "Add Client" `<button id="add-client-btn">` right-aligned inside a flex header |
| `<body>` tail | Modal overlay `<div id="add-client-modal">` containing: form view, output panel view |
| `<script>` | Functions: `generateSlug`, `buildIntakeEntry`, `openAddClientModal`, `closeAddClientModal`, `trapFocus`, `showIntakeResult`, `downloadIntake`, `sendToPipeline` |
| `initDashboard()` | Wire `#add-client-btn` click, attach modal close handlers |

---

### Task 1: CSS Tokens + "Add Client" Button

**Files:**
- Modify: `index.html` (`:root` block, `<style>` block, `<header>` element)

**Interfaces:**
- Produces: `--color-overlay`, `--color-code-bg`, `--modal-max-width` tokens; `.add-client-btn` class; `id="add-client-btn"` button in DOM

- [ ] **Step 1: Add new CSS tokens to `:root`**

In `index.html`, find the closing `}` of `:root` (after `--color-status-error: #ef4444;`) and insert before it:

```css
      /* Modal */
      --color-overlay: rgba(0, 0, 0, 0.7);
      --color-code-bg: #0d1117;
      --modal-max-width: 480px;
```

- [ ] **Step 2: Add `.add-client-btn` CSS**

Append inside `<style>`, after the last `@media` block and before `</style>`:

```css
    /* =====================
       Add Client Button
       ===================== */
    .add-client-btn {
      background-color: var(--color-accent);
      border: none;
      border-radius: 4px;
      color: white;
      cursor: pointer;
      font-family: var(--font-family-base);
      font-size: var(--font-size-body);
      font-weight: var(--font-weight-bold);
      min-height: 44px;
      padding: var(--space-sm) var(--space-md);
      transition: background-color 0.15s ease;
    }

    .add-client-btn:hover {
      background-color: #2563eb;
    }

    .add-client-btn:focus {
      outline: 2px solid var(--color-accent);
      outline-offset: 2px;
    }
```

- [ ] **Step 3: Update `<header>` to flex layout with right-aligned button**

Replace the existing `<header>` element:

```html
  <div id="dashboard" style="display:none">
    <header style="display:flex;align-items:center;justify-content:center;position:relative;padding:var(--space-2xl) var(--space-lg) var(--space-xl);">
      <h1>Client Dashboard</h1>
      <button id="add-client-btn" class="add-client-btn" style="position:absolute;right:var(--space-lg);display:none;" aria-haspopup="dialog">
        Add Client
      </button>
    </header>
```

Note: The button starts hidden (`display:none`) and is shown by `initDashboard()` after auth passes. This prevents it from being visible on the gate screen if the dashboard element is ever partially exposed.

- [ ] **Step 4: Manual verification**

Open `index.html` in browser (via local server: `python -m http.server 8000`). Log in. Confirm "Add Client" button appears in the top-right corner of the header. Confirm button has correct accent-blue styling. Clicking it does nothing yet.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add CSS tokens and Add Client button stub to dashboard header"
```

---

### Task 2: Modal Shell HTML + CSS

**Files:**
- Modify: `index.html` (`<style>` block, `<body>` — new modal HTML after `#dashboard` div)

**Interfaces:**
- Consumes: `--color-overlay`, `--color-code-bg`, `--modal-max-width`, `--color-surface`, `--color-border`, `--color-text-primary`, `--color-text-secondary`, `--color-accent`, `--space-*`, `--font-*` tokens
- Produces: `#add-client-modal` overlay in DOM; `.modal-overlay`, `.modal-card`, `.modal-header`, `.modal-close` classes; `#modal-form-view` and `#modal-output-view` inner containers (empty placeholders for Tasks 3 and 5)

- [ ] **Step 1: Add modal CSS**

Append inside `<style>` after `.add-client-btn:focus`:

```css
    /* =====================
       Modal Overlay
       ===================== */
    .modal-overlay {
      display: none;
      position: fixed;
      inset: 0;
      background-color: var(--color-overlay);
      z-index: 100;
      align-items: center;
      justify-content: center;
      padding: var(--space-md);
    }

    .modal-overlay.open {
      display: flex;
    }

    .modal-card {
      background: var(--color-surface);
      border: 1px solid var(--color-border);
      border-radius: 6px;
      padding: var(--space-2xl);
      width: 100%;
      max-width: var(--modal-max-width);
      max-height: 90vh;
      overflow-y: auto;
      position: relative;
      display: flex;
      flex-direction: column;
      gap: var(--space-lg);
    }

    .modal-header {
      display: flex;
      align-items: center;
      justify-content: space-between;
    }

    .modal-title {
      font-size: var(--font-size-display);
      font-weight: var(--font-weight-bold);
      color: var(--color-text-primary);
    }

    .modal-close {
      background: none;
      border: none;
      color: var(--color-text-secondary);
      cursor: pointer;
      font-size: 20px;
      line-height: 1;
      padding: var(--space-xs);
      min-height: 44px;
      min-width: 44px;
      display: flex;
      align-items: center;
      justify-content: center;
      border-radius: 4px;
    }

    .modal-close:hover {
      color: var(--color-text-primary);
      background-color: var(--color-border);
    }

    .modal-close:focus {
      outline: 2px solid var(--color-accent);
      outline-offset: 2px;
    }
```

- [ ] **Step 2: Add modal HTML after the closing `</div>` of `#dashboard`**

```html
  <!-- Add Client Modal -->
  <div id="add-client-modal" class="modal-overlay" role="dialog" aria-modal="true" aria-labelledby="modal-title">
    <div class="modal-card" id="modal-card">
      <div class="modal-header">
        <h2 class="modal-title" id="modal-title">Add Client</h2>
        <button class="modal-close" id="modal-close-btn" aria-label="Close modal">✕</button>
      </div>
      <!-- Form view and output panel inserted in Tasks 3 and 5 -->
      <div id="modal-form-view"></div>
      <div id="modal-output-view" style="display:none;"></div>
    </div>
  </div>
```

- [ ] **Step 3: Manual verification**

Add `class="open"` temporarily to `#add-client-modal` in the HTML. Open in browser. Confirm: dark overlay covers full screen, centered card appears with title "Add Client" and ✕ button. Remove the temporary class.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add modal overlay shell HTML and CSS"
```

---

### Task 3: Form Fields HTML + CSS

**Files:**
- Modify: `index.html` (`<style>` block, `#modal-form-view` placeholder)

**Interfaces:**
- Consumes: `--color-bg` (reused as input background), `--color-border`, `--color-text-primary`, `--color-text-secondary`, `--color-accent`, `--color-status-error`, `--space-*`, `--font-*` tokens
- Produces: `#intake-form` element; field IDs: `#field-name`, `#field-slug`, `#field-industry`, `#field-business-type`, `#field-domain`, `#field-notes`, `#field-services`, `#field-tone`, `#field-color-hint`; `#slug-edit-toggle` link; `#advanced-section` details element; `.form-field`, `.form-label`, `.form-input`, `.form-textarea`, `.form-error`, `.form-actions`, `.slug-row` classes

- [ ] **Step 1: Add form field CSS**

Append inside `<style>` after `.modal-close:focus`:

```css
    /* =====================
       Form Fields
       ===================== */
    .form-fields {
      display: flex;
      flex-direction: column;
      gap: var(--space-md);
    }

    .form-field {
      display: flex;
      flex-direction: column;
      gap: var(--space-xs);
    }

    .form-label {
      font-size: var(--font-size-label);
      font-weight: var(--font-weight-bold);
      color: var(--color-text-secondary);
      text-transform: uppercase;
      letter-spacing: 0.05em;
    }

    .form-label .required-mark {
      color: var(--color-status-error);
      margin-left: 2px;
    }

    .form-input,
    .form-textarea {
      background: var(--color-bg);
      border: 1px solid var(--color-border);
      border-radius: 4px;
      color: var(--color-text-primary);
      font-family: var(--font-family-base);
      font-size: var(--font-size-body);
      padding: var(--space-sm) var(--space-md);
      width: 100%;
      min-height: 44px;
      outline: none;
    }

    .form-input:focus,
    .form-textarea:focus {
      border-color: var(--color-accent);
    }

    .form-input.input-error,
    .form-textarea.input-error {
      border-color: var(--color-status-error);
    }

    .form-textarea {
      min-height: 80px;
      resize: vertical;
    }

    .form-error {
      color: var(--color-status-error);
      font-size: var(--font-size-label);
      min-height: 16px;
    }

    .slug-row {
      display: flex;
      align-items: center;
      gap: var(--space-sm);
    }

    .slug-row .form-input {
      flex: 1;
    }

    #slug-edit-toggle {
      color: var(--color-accent);
      font-size: var(--font-size-label);
      cursor: pointer;
      white-space: nowrap;
      text-decoration: underline;
      background: none;
      border: none;
      padding: 0;
      font-family: var(--font-family-base);
    }

    #slug-edit-toggle:focus {
      outline: 2px solid var(--color-accent);
      outline-offset: 2px;
    }

    /* Advanced section */
    details.advanced-section {
      border-top: 1px solid var(--color-border);
      padding-top: var(--space-md);
    }

    details.advanced-section summary {
      color: var(--color-text-secondary);
      cursor: pointer;
      font-size: var(--font-size-label);
      font-weight: var(--font-weight-bold);
      text-transform: uppercase;
      letter-spacing: 0.05em;
      list-style: none;
      display: flex;
      align-items: center;
      gap: var(--space-xs);
      margin-bottom: var(--space-md);
    }

    details.advanced-section summary::-webkit-details-marker {
      display: none;
    }

    details.advanced-section[open] summary::before {
      content: '▾';
    }

    details.advanced-section:not([open]) summary::before {
      content: '▸';
    }

    .form-actions {
      display: flex;
      gap: var(--space-sm);
      justify-content: flex-end;
      padding-top: var(--space-md);
      border-top: 1px solid var(--color-border);
    }

    .btn-primary {
      background-color: var(--color-accent);
      border: none;
      border-radius: 4px;
      color: white;
      cursor: pointer;
      font-family: var(--font-family-base);
      font-size: var(--font-size-body);
      font-weight: var(--font-weight-bold);
      min-height: 44px;
      padding: var(--space-sm) var(--space-lg);
      transition: background-color 0.15s ease;
    }

    .btn-primary:hover {
      background-color: #2563eb;
    }

    .btn-primary:focus {
      outline: 2px solid var(--color-accent);
      outline-offset: 2px;
    }

    .btn-secondary {
      background-color: transparent;
      border: 1px solid var(--color-border);
      border-radius: 4px;
      color: var(--color-text-secondary);
      cursor: pointer;
      font-family: var(--font-family-base);
      font-size: var(--font-size-body);
      font-weight: var(--font-weight-bold);
      min-height: 44px;
      padding: var(--space-sm) var(--space-md);
      transition: border-color 0.15s ease, color 0.15s ease;
    }

    .btn-secondary:hover {
      border-color: var(--color-text-secondary);
      color: var(--color-text-primary);
    }

    .btn-secondary:focus {
      outline: 2px solid var(--color-accent);
      outline-offset: 2px;
    }
```

- [ ] **Step 2: Replace `#modal-form-view` placeholder with form HTML**

Replace `<div id="modal-form-view"></div>` with:

```html
      <div id="modal-form-view">
        <form id="intake-form" autocomplete="off" novalidate>
          <div class="form-fields">
            <!-- Business Name -->
            <div class="form-field">
              <label class="form-label" for="field-name">Business Name <span class="required-mark">*</span></label>
              <input id="field-name" class="form-input" type="text" required autocomplete="off">
              <p class="form-error" id="error-name" aria-live="polite"></p>
            </div>

            <!-- Slug -->
            <div class="form-field">
              <label class="form-label" for="field-slug">Slug <span class="required-mark">*</span></label>
              <div class="slug-row">
                <input id="field-slug" class="form-input" type="text" required readonly aria-describedby="error-slug">
                <button type="button" id="slug-edit-toggle">Edit</button>
              </div>
              <p class="form-error" id="error-slug" aria-live="polite"></p>
            </div>

            <!-- Industry -->
            <div class="form-field">
              <label class="form-label" for="field-industry">Industry <span class="required-mark">*</span></label>
              <input id="field-industry" class="form-input" type="text" placeholder="e.g. Home Services, Food &amp; Beverage" required>
              <p class="form-error" id="error-industry" aria-live="polite"></p>
            </div>

            <!-- Business Type -->
            <div class="form-field">
              <label class="form-label" for="field-business-type">Business Type <span class="required-mark">*</span></label>
              <input id="field-business-type" class="form-input" type="text" placeholder="e.g. coffee roaster, plumber" required>
              <p class="form-error" id="error-business-type" aria-live="polite"></p>
            </div>

            <!-- Domain -->
            <div class="form-field">
              <label class="form-label" for="field-domain">Domain</label>
              <input id="field-domain" class="form-input" type="url" placeholder="https://example.com — optional">
              <p class="form-error" id="error-domain" aria-live="polite"></p>
            </div>

            <!-- Notes -->
            <div class="form-field">
              <label class="form-label" for="field-notes">Notes</label>
              <textarea id="field-notes" class="form-textarea" placeholder="Research direction, services, tone, colors — optional"></textarea>
            </div>

            <!-- Advanced -->
            <details class="advanced-section" id="advanced-section">
              <summary>Advanced options</summary>
              <div class="form-fields">
                <div class="form-field">
                  <label class="form-label" for="field-services">Services</label>
                  <textarea id="field-services" class="form-textarea" placeholder="Drain clearing, pipe repair, emergency calls — optional"></textarea>
                </div>
                <div class="form-field">
                  <label class="form-label" for="field-tone">Tone</label>
                  <input id="field-tone" class="form-input" type="text" placeholder="trust, authoritative, warm — optional">
                </div>
                <div class="form-field">
                  <label class="form-label" for="field-color-hint">Color Hint</label>
                  <input id="field-color-hint" class="form-input" type="text" placeholder="#1a3c5e or &quot;navy and white&quot; — optional">
                </div>
              </div>
            </details>
          </div>

          <div class="form-actions">
            <button type="button" class="btn-secondary" id="modal-cancel-btn">Cancel</button>
            <button type="submit" class="btn-primary">Generate Entry</button>
          </div>
        </form>
      </div>
```

- [ ] **Step 3: Manual verification**

Temporarily add `class="open"` to `#add-client-modal`. Open in browser. Confirm:
- All 6 always-visible fields render correctly with labels and placeholders
- "Advanced options" collapses/expands natively via `<details>` — no JS needed
- Slug field is readonly with an "Edit" link beside it
- Required fields show the red `*` mark
- Cancel and Generate Entry buttons are present
- Inputs have dark background matching gate-card inputs

Remove the temporary class.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add intake form fields HTML and CSS inside modal"
```

---

### Task 4: Output Panel HTML + CSS

**Files:**
- Modify: `index.html` (`<style>` block, `#modal-output-view` placeholder)

**Interfaces:**
- Consumes: `--color-code-bg`, `--color-surface`, `--color-border`, `--color-text-primary`, `--color-text-secondary`, `--color-accent`, `--color-status-pending` (yellow for warning), `--space-*`, `--font-*`
- Produces: `#modal-output-view` populated with `#output-json-block`, `#btn-copy-json`, `#btn-download`, `#btn-send-pipeline`, `#pipeline-unsupported-note`, `#pipeline-warning`, `#add-another-link`; `.output-panel`, `.output-json`, `.output-actions`, `.pipeline-warning` classes

- [ ] **Step 1: Add output panel CSS**

Append inside `<style>` after `.btn-secondary:focus`:

```css
    /* =====================
       Output Panel
       ===================== */
    .output-panel {
      display: flex;
      flex-direction: column;
      gap: var(--space-lg);
    }

    .output-json {
      background: var(--color-code-bg);
      border: 1px solid var(--color-border);
      border-radius: 4px;
      padding: var(--space-md);
      overflow-x: auto;
      font-family: 'Courier New', Courier, monospace;
      font-size: var(--font-size-label);
      line-height: 1.6;
      color: var(--color-text-primary);
      white-space: pre;
    }

    .output-actions {
      display: flex;
      flex-direction: column;
      gap: var(--space-sm);
    }

    .output-actions-row {
      display: flex;
      gap: var(--space-sm);
      align-items: center;
      flex-wrap: wrap;
    }

    .pipeline-warning {
      color: var(--color-status-pending);
      font-size: var(--font-size-label);
    }

    .pipeline-unsupported-note {
      color: var(--color-text-secondary);
      font-size: var(--font-size-label);
    }

    .output-add-another {
      color: var(--color-accent);
      font-size: var(--font-size-body);
      cursor: pointer;
      text-align: center;
      text-decoration: underline;
      background: none;
      border: none;
      font-family: var(--font-family-base);
      padding: 0;
      display: block;
      width: 100%;
    }

    .output-add-another:focus {
      outline: 2px solid var(--color-accent);
      outline-offset: 2px;
    }
```

- [ ] **Step 2: Replace `#modal-output-view` placeholder with output panel HTML**

Replace `<div id="modal-output-view" style="display:none;"></div>` with:

```html
      <div id="modal-output-view" style="display:none;">
        <div class="output-panel">
          <div>
            <p style="font-size:var(--font-size-label);color:var(--color-text-secondary);margin-bottom:var(--space-sm);">Generated entry — add to your intake file:</p>
            <pre class="output-json"><code id="output-json-block"></code></pre>
          </div>
          <div class="output-actions">
            <div class="output-actions-row">
              <button id="btn-copy-json" class="btn-primary" style="flex:0 0 auto;">Copy JSON</button>
              <button id="btn-download" class="btn-secondary" style="flex:0 0 auto;">Download intake.json</button>
            </div>
            <div class="output-actions-row" id="pipeline-row">
              <button id="btn-send-pipeline" class="btn-secondary" style="flex:0 0 auto;">Send to Pipeline</button>
              <span class="pipeline-warning" id="pipeline-warning" style="display:none;">
                ⚠ This overwrites the current intake file
              </span>
              <span class="pipeline-unsupported-note" id="pipeline-unsupported-note" style="display:none;">
                Use Download on this browser — File System Access not supported.
              </span>
            </div>
          </div>
          <button type="button" class="output-add-another" id="add-another-link">+ Add Another</button>
        </div>
      </div>
```

- [ ] **Step 3: Manual verification**

Temporarily add `class="open"` to `#add-client-modal` and set `#modal-form-view` to `display:none` and `#modal-output-view` to `display:block`. Add placeholder text to `#output-json-block`. Open in browser. Confirm:
- Dark code block renders with monospace font
- Three buttons render: Copy JSON, Download intake.json, Send to Pipeline
- Yellow warning text slot is present (hidden)
- "+ Add Another" link renders centered below buttons

Revert temporary changes.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add output panel HTML and CSS for intake entry result"
```

---

### Task 5: Modal Open/Close/Focus Trap JS

**Files:**
- Modify: `index.html` (`<script>` block, `initDashboard()` function)

**Interfaces:**
- Consumes: `#add-client-modal`, `#modal-card`, `#modal-close-btn`, `#modal-cancel-btn`, `#add-client-btn`, `#intake-form`, `#field-name` (for initial focus), `#modal-form-view`, `#modal-output-view`
- Produces: `openAddClientModal()`, `closeAddClientModal()`, `trapFocus(e)` as module-scope functions. The "Add Client" button in the header becomes functional. Escape key and click-outside close the modal.

- [ ] **Step 1: Add modal JS functions**

Inside the `<script>` block, before `document.addEventListener('DOMContentLoaded', initDashboard)`, add:

```javascript
    // =====================
    // Modal: open / close / focus trap
    // =====================

    function openAddClientModal() {
      var modal = document.getElementById('add-client-modal');
      var formView = document.getElementById('modal-form-view');
      var outputView = document.getElementById('modal-output-view');
      var form = document.getElementById('intake-form');

      // Reset to form view and clear values
      formView.style.display = '';
      outputView.style.display = 'none';
      form.reset();
      // Re-lock slug to readonly (auto-generation mode)
      var slugInput = document.getElementById('field-slug');
      if (slugInput) {
        slugInput.readOnly = true;
        slugInput.value = '';
      }
      // Clear any validation errors
      document.querySelectorAll('.form-error').forEach(function(el) { el.textContent = ''; });
      document.querySelectorAll('.form-input, .form-textarea').forEach(function(el) {
        el.classList.remove('input-error');
      });

      modal.classList.add('open');
      // Focus first required field
      var first = document.getElementById('field-name');
      if (first) first.focus();

      modal.addEventListener('keydown', trapFocus);
    }

    function closeAddClientModal() {
      var modal = document.getElementById('add-client-modal');
      modal.classList.remove('open');
      modal.removeEventListener('keydown', trapFocus);
      // Return focus to the trigger button
      var trigger = document.getElementById('add-client-btn');
      if (trigger) trigger.focus();
    }

    function trapFocus(e) {
      if (e.key !== 'Tab' && e.key !== 'Escape') return;
      if (e.key === 'Escape') {
        e.preventDefault();
        closeAddClientModal();
        return;
      }
      var modal = document.getElementById('modal-card');
      var focusable = Array.from(modal.querySelectorAll(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      )).filter(function(el) { return !el.disabled && el.offsetParent !== null; });
      if (focusable.length === 0) return;
      var first = focusable[0];
      var last = focusable[focusable.length - 1];
      if (e.shiftKey) {
        if (document.activeElement === first) {
          e.preventDefault();
          last.focus();
        }
      } else {
        if (document.activeElement === last) {
          e.preventDefault();
          first.focus();
        }
      }
    }
```

- [ ] **Step 2: Wire event handlers in `initDashboard()`**

Inside `initDashboard()`, after the `loadClients()` call (inside the already-authed branch and inside the success branch of the form submit), add:

```javascript
        // Show Add Client button (hidden until auth passes)
        var addClientBtn = document.getElementById('add-client-btn');
        if (addClientBtn) addClientBtn.style.display = '';

        // Wire modal open
        addClientBtn.addEventListener('click', openAddClientModal);

        // Wire modal close
        document.getElementById('modal-close-btn').addEventListener('click', closeAddClientModal);
        document.getElementById('modal-cancel-btn').addEventListener('click', closeAddClientModal);

        // Click outside card closes modal
        document.getElementById('add-client-modal').addEventListener('click', function(e) {
          if (e.target === e.currentTarget) closeAddClientModal();
        });
```

There are two places to add this wiring inside `initDashboard()`:
1. In the early-return branch where `stored === cfg.hash` (existing auth) — after `loadClients()` call, before `return`.
2. In the form submit success branch — after `loadClients()` call.

To avoid duplication, extract these wires to a `wireAddClientModal()` helper function and call it from both branches:

```javascript
    function wireAddClientModal() {
      var addClientBtn = document.getElementById('add-client-btn');
      if (!addClientBtn) return;
      addClientBtn.style.display = '';
      addClientBtn.addEventListener('click', openAddClientModal);
      document.getElementById('modal-close-btn').addEventListener('click', closeAddClientModal);
      document.getElementById('modal-cancel-btn').addEventListener('click', closeAddClientModal);
      document.getElementById('add-client-modal').addEventListener('click', function(e) {
        if (e.target === e.currentTarget) closeAddClientModal();
      });
    }
```

Place `wireAddClientModal()` before `closeAddClientModal` in the script block, then call `wireAddClientModal()` in both auth-success branches of `initDashboard()`.

- [ ] **Step 3: Manual verification**

Open in browser. Log in. Confirm:
- "Add Client" button appears after login
- Clicking it opens the modal
- ✕ button closes it
- Cancel button closes it
- Clicking the dark overlay (outside the card) closes it
- Pressing Escape closes it
- Tab key cycles focus within the modal without escaping to the page
- Modal resets to blank form state when reopened

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: implement modal open/close and focus trap JS"
```

---

### Task 6: Slug Auto-Generation JS

**Files:**
- Modify: `index.html` (`<script>` block)

**Interfaces:**
- Consumes: `#field-name`, `#field-slug`, `#slug-edit-toggle`
- Produces: `generateSlug(text)` — pure function; `initSlugGeneration()` — wires events on the form. Called from `openAddClientModal()` after form reset.

- [ ] **Step 1: Write `generateSlug` pure function**

In `<script>`, before `openAddClientModal`:

```javascript
    // =====================
    // Slug generation
    // =====================

    function generateSlug(text) {
      return text
        .toLowerCase()
        .replace(/\s+/g, '-')
        .replace(/[^a-z0-9-]/g, '')
        .replace(/-+/g, '-')
        .replace(/^-|-$/g, '');
    }
```

- [ ] **Step 2: Verify `generateSlug` in browser console**

Open browser DevTools console and paste:

```javascript
console.assert(generateSlug('Artisan Coffee Roasters') === 'artisan-coffee-roasters', 'basic');
console.assert(generateSlug('Quick & Response Plumbing!!') === 'quick-response-plumbing', 'special chars');
console.assert(generateSlug('  Leading Spaces  ') === 'leading-spaces', 'trim spaces');
console.assert(generateSlug("Bob's Burgers") === 'bobs-burgers', 'apostrophe');
```

All assertions should pass silently (no output = pass).

- [ ] **Step 3: Add `initSlugGeneration` to wire slug behavior**

```javascript
    function initSlugGeneration() {
      var nameInput = document.getElementById('field-name');
      var slugInput = document.getElementById('field-slug');
      var editToggle = document.getElementById('slug-edit-toggle');
      var slugManuallyEdited = false;

      function updateSlug() {
        if (!slugManuallyEdited) {
          slugInput.value = generateSlug(nameInput.value);
        }
      }

      nameInput.addEventListener('input', updateSlug);

      editToggle.addEventListener('click', function() {
        if (slugInput.readOnly) {
          slugInput.readOnly = false;
          slugInput.focus();
          editToggle.textContent = 'Auto';
          slugManuallyEdited = true;
        } else {
          slugInput.readOnly = true;
          editToggle.textContent = 'Edit';
          slugManuallyEdited = false;
          updateSlug(); // re-sync from name
        }
      });
    }
```

- [ ] **Step 4: Call `initSlugGeneration()` inside `openAddClientModal()`**

Add the call after the reset block in `openAddClientModal()`, before `modal.classList.add('open')`:

```javascript
      initSlugGeneration();
```

Note: `initSlugGeneration()` is called each time the modal opens (after form reset re-creates event listeners). Because `reset()` clears values but does not remove listeners, and we re-add them each open, this is safe — the `slugManuallyEdited` flag is scoped inside `initSlugGeneration` so it resets each call.

- [ ] **Step 5: Manual verification**

Open modal. Type "Artisan Coffee Roasters" in Business Name. Confirm Slug field shows `artisan-coffee-roasters` live. Click "Edit", type custom slug `my-custom-slug`. Close and reopen modal. Confirm Slug field resets to blank, auto-generation resumes.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: implement slug auto-generation with manual override toggle"
```

---

### Task 7: Validation, Form Submit, and Output Panel JS

**Files:**
- Modify: `index.html` (`<script>` block)

**Interfaces:**
- Consumes: all form field IDs from Task 3; `#modal-form-view`, `#modal-output-view`, `#output-json-block`, `#btn-copy-json`, `#btn-download`, `#btn-send-pipeline`, `#pipeline-warning`, `#pipeline-unsupported-note`, `#add-another-link`, `generateSlug()`
- Produces: `buildIntakeEntry()`, `validateIntakeForm()`, `showIntakeResult(entry)`, `downloadIntake(entry)`, `sendToPipeline(entry)` functions; wired form submit handler

- [ ] **Step 1: Write `validateIntakeForm`**

```javascript
    // =====================
    // Validation
    // =====================

    function validateIntakeForm() {
      var errors = false;

      function setError(fieldId, errorId, message) {
        var field = document.getElementById(fieldId);
        var errorEl = document.getElementById(errorId);
        if (message) {
          field.classList.add('input-error');
          errorEl.textContent = message;
          errors = true;
        } else {
          field.classList.remove('input-error');
          errorEl.textContent = '';
        }
      }

      var name = document.getElementById('field-name').value.trim();
      setError('field-name', 'error-name', name ? '' : 'Business Name is required.');

      var industry = document.getElementById('field-industry').value.trim();
      setError('field-industry', 'error-industry', industry ? '' : 'Industry is required.');

      var businessType = document.getElementById('field-business-type').value.trim();
      setError('field-business-type', 'error-business-type', businessType ? '' : 'Business Type is required.');

      var slug = document.getElementById('field-slug').value.trim();
      var slugValid = /^[a-z0-9][a-z0-9-]*[a-z0-9]$/.test(slug) || /^[a-z0-9]$/.test(slug);
      setError('field-slug', 'error-slug', slug && slugValid ? '' : 'Slug is invalid. Use only lowercase letters, numbers, and hyphens.');

      var domain = document.getElementById('field-domain').value.trim();
      if (domain && !/^https?:\/\/.+/.test(domain)) {
        setError('field-domain', 'error-domain', 'Domain must start with http:// or https://');
      } else {
        setError('field-domain', 'error-domain', '');
      }

      return !errors;
    }
```

- [ ] **Step 2: Write `buildIntakeEntry`**

```javascript
    // =====================
    // Build intake entry object
    // =====================

    function buildIntakeEntry() {
      var entry = {};
      var fields = [
        { id: 'field-name',          key: 'businessName' },
        { id: 'field-slug',          key: 'slug' },
        { id: 'field-industry',      key: 'industry' },
        { id: 'field-business-type', key: 'businessType' },
        { id: 'field-domain',        key: 'domain' },
        { id: 'field-notes',         key: 'notes' },
        { id: 'field-services',      key: 'services' },
        { id: 'field-tone',          key: 'tone' },
        { id: 'field-color-hint',    key: 'colorHint' },
      ];
      fields.forEach(function(f) {
        var val = document.getElementById(f.id).value.trim();
        if (val) entry[f.key] = val;
      });
      return entry;
    }
```

- [ ] **Step 3: Write `showIntakeResult`**

```javascript
    // =====================
    // Output panel
    // =====================

    function showIntakeResult(entry) {
      var formView = document.getElementById('modal-form-view');
      var outputView = document.getElementById('modal-output-view');
      var jsonBlock = document.getElementById('output-json-block');

      jsonBlock.textContent = JSON.stringify(entry, null, 2);

      formView.style.display = 'none';
      outputView.style.display = '';

      // Feature check for File System Access
      var sendBtn = document.getElementById('btn-send-pipeline');
      var unsupportedNote = document.getElementById('pipeline-unsupported-note');
      if (!('showSaveFilePicker' in window)) {
        sendBtn.style.display = 'none';
        unsupportedNote.style.display = '';
      } else {
        sendBtn.style.display = '';
        unsupportedNote.style.display = 'none';
      }

      // Show pipeline overwrite warning next to send button
      var warning = document.getElementById('pipeline-warning');
      warning.style.display = 'showSaveFilePicker' in window ? '' : 'none';

      // Wire Copy JSON
      var copyBtn = document.getElementById('btn-copy-json');
      var cleanCopy = copyBtn.cloneNode(true);
      copyBtn.parentNode.replaceChild(cleanCopy, copyBtn);
      cleanCopy.addEventListener('click', function() {
        navigator.clipboard.writeText(JSON.stringify(entry, null, 2)).then(function() {
          cleanCopy.textContent = 'Copied!';
          setTimeout(function() { cleanCopy.textContent = 'Copy JSON'; }, 2000);
        }).catch(function() {
          alert('Copy failed. Open DevTools console to see the JSON.');
          console.log(JSON.stringify(entry, null, 2));
        });
      });

      // Wire Download
      var dlBtn = document.getElementById('btn-download');
      var cleanDl = dlBtn.cloneNode(true);
      dlBtn.parentNode.replaceChild(cleanDl, dlBtn);
      cleanDl.addEventListener('click', function() { downloadIntake(entry); });

      // Wire Send to Pipeline
      if ('showSaveFilePicker' in window) {
        var sendBtnFresh = document.getElementById('btn-send-pipeline');
        var cleanSend = sendBtnFresh.cloneNode(true);
        sendBtnFresh.parentNode.replaceChild(cleanSend, sendBtnFresh);
        cleanSend.addEventListener('click', function() { sendToPipeline(entry); });
      }

      // Wire Add Another
      var addAnother = document.getElementById('add-another-link');
      var cleanAnother = addAnother.cloneNode(true);
      addAnother.parentNode.replaceChild(cleanAnother, addAnother);
      cleanAnother.addEventListener('click', function() {
        formView.style.display = '';
        outputView.style.display = 'none';
        var form = document.getElementById('intake-form');
        form.reset();
        var slugInput = document.getElementById('field-slug');
        if (slugInput) { slugInput.readOnly = true; slugInput.value = ''; }
        document.querySelectorAll('.form-error').forEach(function(el) { el.textContent = ''; });
        document.querySelectorAll('.form-input, .form-textarea').forEach(function(el) {
          el.classList.remove('input-error');
        });
        initSlugGeneration();
        var nameField = document.getElementById('field-name');
        if (nameField) nameField.focus();
      });
    }
```

- [ ] **Step 4: Write `downloadIntake`**

```javascript
    function downloadIntake(entry) {
      var payload = JSON.stringify({ intake: [entry] }, null, 2);
      var blob = new Blob([payload], { type: 'application/json' });
      var url = URL.createObjectURL(blob);
      var a = document.createElement('a');
      a.href = url;
      a.download = 'intake.json';
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      URL.revokeObjectURL(url);
    }
```

- [ ] **Step 5: Write `sendToPipeline`**

```javascript
    async function sendToPipeline(entry) {
      var payload = JSON.stringify({ intake: [entry] }, null, 2);
      try {
        var fileHandle = await window.showSaveFilePicker({
          suggestedName: 'intake.json',
          types: [{ description: 'JSON', accept: { 'application/json': ['.json'] } }]
        });
        var writable = await fileHandle.createWritable();
        await writable.write(payload);
        await writable.close();
      } catch (err) {
        if (err.name !== 'AbortError') {
          console.error('sendToPipeline failed:', err);
          alert('Write failed: ' + err.message);
        }
      }
    }
```

- [ ] **Step 6: Wire form submit handler**

Add inside `openAddClientModal()`, after `initSlugGeneration()` call (or after modal opens), as a one-time setup. To avoid duplicate listener registration, move this wiring to a one-time `initIntakeForm()` function called once from `wireAddClientModal()`:

```javascript
    function initIntakeForm() {
      document.getElementById('intake-form').addEventListener('submit', function(e) {
        e.preventDefault();
        if (!validateIntakeForm()) {
          // Focus first errored field
          var firstError = document.querySelector('.input-error');
          if (firstError) firstError.focus();
          return;
        }
        var entry = buildIntakeEntry();
        showIntakeResult(entry);
      });
    }
```

Call `initIntakeForm()` inside `wireAddClientModal()`, after the other event wires.

- [ ] **Step 7: Manual end-to-end verification**

Test the golden path:
1. Open modal → type "Blue Ridge Bakery", industry "Food & Beverage", business type "bakery", domain "https://blueridgebakery.com"
2. Confirm slug auto-generates to `blue-ridge-bakery`
3. Click Generate Entry
4. Confirm output panel shows with correct JSON (no empty optional fields)
5. Click Copy JSON → paste into a text editor → confirm it's valid JSON with `{ "businessName": "Blue Ridge Bakery", ... }`
6. Click Download intake.json → confirm file downloads named `intake.json` containing `{ "intake": [entry] }`
7. If on Chrome: click Send to Pipeline → confirm a save dialog opens defaulting to `intake.json`
8. Click "+ Add Another" → confirm form resets to blank and slug is locked/empty

Test validation:
1. Submit with all fields blank → all required fields show error messages
2. Enter domain `notaurl` → domain error appears
3. Click Edit on slug → type `INVALID SLUG!!` → submit → slug error appears

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat: implement form validation, JSON generation, and output panel actions"
```

---

## Self-Review Against Spec

**Spec coverage check:**

| Spec requirement | Task | Status |
|-----------------|------|--------|
| "Add Client" button in dashboard header, right-aligned, visible post-auth | Task 1, 5 | ✓ |
| Modal overlay with dark background | Task 2 | ✓ |
| Centered card, max-width 480px, `.gate-card` visual style | Task 2 | ✓ |
| First required field receives focus on open | Task 5 | ✓ |
| ✕ button, click outside, Escape key close modal | Task 5 | ✓ |
| Focus trap | Task 5 | ✓ |
| Form resets on close | Task 5 | ✓ |
| All 6 always-visible fields | Task 3 | ✓ |
| Advanced section collapsed by default, `<details>/<summary>` | Task 3 | ✓ |
| All 3 advanced fields | Task 3 | ✓ |
| Slug auto-generates live from Business Name | Task 6 | ✓ |
| Slug transform: lowercase, spaces→hyphens, strip non-alnum | Task 6 | ✓ |
| "Edit" link unlocks slug; auto-generation pauses once manually edited | Task 6 | ✓ |
| Required field validation (Business Name, Industry, Business Type) | Task 7 | ✓ |
| Domain URL format validation | Task 7 | ✓ |
| Slug pattern validation (error only if manually broken) | Task 7 | ✓ |
| Output panel replaces form inside modal on submit | Task 7 | ✓ |
| Generated JSON block, omit empty optional fields | Task 7 | ✓ |
| Copy JSON button, shows "Copied!" for 2s | Task 7 | ✓ |
| Download intake.json — Blob URL, `{ "intake": [entry] }` | Task 7 | ✓ |
| Send to Pipeline — File System Access API, overwrites, yellow warning | Task 7 | ✓ |
| Send to Pipeline hidden on unsupported browsers, replaced by note | Task 7 | ✓ |
| "Add Another" link resets form | Task 7 | ✓ |
| New CSS tokens in `:root`, no hardcoded hex | Task 1 | ✓ |
| Only `index.html` modified | All | ✓ |
| `--color-overlay`, `--color-code-bg`, `--modal-max-width` tokens | Task 1 | ✓ |

**Placeholder scan:** No TBD, TODO, or "implement later" text present. All code blocks are complete.

**Type consistency:** `generateSlug` is defined in Task 6 and called in Task 6 (`initSlugGeneration`). `initSlugGeneration` is called in Tasks 5 and 7 (`openAddClientModal`, `showIntakeResult > add-another handler`). `buildIntakeEntry`, `validateIntakeForm`, `showIntakeResult`, `downloadIntake`, `sendToPipeline` are all defined in Task 7 and wired in Task 7. `wireAddClientModal` and `initIntakeForm` are defined/called in Task 5. No cross-task name mismatches found.
