# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **static, client-side web app** for a Pacific Office Automation (POA) field service technician. It consists of two cooperating single-file HTML apps deployed via GitHub Pages at `https://charleshuff.github.io/POA/`.

There is no build step, no package manager, no test framework, and no linting. Files are served as-is.

## Development

**To run locally:** Open any `.html` file directly in a browser, or use a simple HTTP server (required for clipboard APIs to work over `localhost` vs. `file://`):

```sh
python3 -m http.server 8080
# then open http://localhost:8080
```

**To deploy:** Push to `main` — GitHub Pages serves the repo root automatically.

## Architecture

### Two-Page Workflow

1. **`index.html`** — Call List Parser: the technician pastes raw HTML copied from the ETMS web portal (`etms.pacificoffice.com`). The app parses it with `DOMParser` + CSS selectors + regex to produce structured "call cards".

2. **`call_notes.html`** — Equipment Replacement Info: the technician clicks "Copy & Open Call Notes" on a card, which copies that call's fields to the clipboard and opens `call_notes.html`. The "Paste Calls" button auto-fills the form. After filling the rest, the tech generates a text summary and/or a filled PDF.

Pages communicate **only through the clipboard** — there is no shared state, localStorage, or URL parameters between them.

### Per-File Conventions (both files share the same pattern)

- **Zero external dependencies** in `index.html`. `call_notes.html` loads `pdf-lib@1.17.1` from unpkg CDN (exposed as global `PDFLib`) for PDF form-filling.
- **IIFE-wrapped script** `(function() { ... })()` to isolate scope.
- **`sessionStorage` persistence**: every field auto-saves on `input`/`change` and restores on page load. Radio buttons use key prefix `radio_<name>`. Call card data stored under `callParserData`.
- **Toast notifications**: fixed bottom-center, auto-hide at 2.2s, used for all user feedback.
- **`escapeHTML(str)`**: sanitizes user-sourced strings via `div.textContent` → `div.innerHTML` to prevent XSS.

### `index.html` Parse Logic

- Tech name extracted from `#settingsPanel > strong` (format: `Full Name (TechCode)`).
- Per call (`li[data-call-id]`): customer from `<h3>`, contact from `<p>` with `<strong>` (excludes pipes, `Description:`, `Equipment Note:`, `Hours:`, `Item No.(Model):`), city from `<p>` containing `|` (format: `Street | City, ST ZIP`), equipment from regex `/Item No\.\(Model\):\s*(.*?)\s*Serial:\s*(.*?)\s*Equipment:\s*(.*)/`.

### `call_notes.html` PDF Filling

- Fetches `net_form_with_text_feilds.pdf` (note intentional filename misspelling), fills named form fields via `pdf-lib`, does **not** flatten, downloads as `Network_Install_{customer}_{equipID}.pdf`.

### `testing.html`

A staging variant of `call_notes.html` used for experimenting with UI changes before promoting them. Current difference: the "MPS" field is a `<select>` dropdown instead of Yes/No radio buttons.

## Specification Files

The `.org` files are Org-mode prompt/spec documents that describe the intended behavior of each page. They serve as the source of truth for LLM-assisted regeneration:

- **`call_parser.org`** — spec for `index.html` (selectors, regex, fields, styling, sample data)
- **`call_notes.org`** — spec for `call_notes.html` (all sections, field definitions, summary format, PDF field mappings)
- **`todo.org`** — pending feature work

When making significant changes, update the corresponding `.org` spec to stay in sync.
