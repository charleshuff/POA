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
- **Toast notifications**: fixed bottom-center, auto-hide at 2.2s, used for all user feedback. Dark background `#2d3436`, slides up with 0.3s opacity transition, `pointer-events: none`.
- **`escapeHTML(str)`**: sanitizes user-sourced strings via `div.textContent` → `div.innerHTML` to prevent XSS.

### `testing.html`

A staging variant of `call_notes.html` used for experimenting with UI changes before promoting them. Current difference: the "MPS" field is a `<select>` dropdown instead of Yes/No radio buttons.

---

## Pending Features (`todo.org`)

- **MPS dropdown** (in progress in `testing.html`): options — None (+ Jot Form link), Fixed broken connector, MPS already reporting, Installed MPS.
- **Remote monitoring vendor dropdown**: selecting a vendor shows a clickable link for that service.
- **Point of Contact persistence**: POC should not change between machines for the same customer.

---

## `index.html` — Call List Parser: Full Specification

### ETMS Source HTML Structure

**Technician name** (global, one per page):
```html
<div data-role="panel" id="settingsPanel" data-position="right">
    <p><strong>Charles Huff (20SR20) </strong></p>
</div>
```
- Format: `Full Name (TechCode)` — extract only the name before the parenthesis.
- Fallback: if `#settingsPanel` not found, use the text of `<a href="#settingsPanel">`.

**Each call** is `<li data-call-id="...">` containing an `<a>` with the call content.

### Fields Extracted Per Call

| Field | Source |
|---|---|
| Customer | `<h3>` inside the `<li>` |
| Contact | `<p>` with a `<strong>` child that does NOT contain `\|`, `Description:`, `Equipment Note:`, `Hours:`, or `Item No.(Model):`. If the strong text matches `/^[\s\-]+$/`, use `"N/A"`. |
| City | `<p>` containing `\|` but NOT `"Call Type"`. Format: `Street \| City, ST ZIP`. Split on `\|`, take second part, match `/^([A-Za-z\s.'-]+),/`. |
| Item No. (Model) | Regex capture group 1 from `/Item No\.\(Model\):\s*(.*?)\s*Serial:\s*(.*?)\s*Equipment:\s*(.*)/` |
| Serial Number | Capture group 2 from same regex |
| Equipment ID | Capture group 3 from same regex |
| Technician | Global value from `#settingsPanel` — same for every call |

### Clipboard Text Format (copied per call)

```
Title (Customer): Columbia Hospitality
Contact: CHAD FARHINGER
Technician: Charles Huff
City: Seattle
Item No. (Model): CB751I
Serial Number: ADXD011005724
Equipment ID: 8C28041
```

### Expected Parse Results (reference dataset)

| # | Customer | Contact | City | Model | Serial | Equipment ID |
|---|---|---|---|---|---|---|
| 1 | Columbia Hospitality | Brad Bowler | Seattle | CB751I | ADXD011005724 | 8C28041 |
| 2 | Columbia Hospitality | N/A | Seattle | P60155 | PHNCT8P0XH | 8C28042 |
| 3 | Columbia Hospitality | N/A | Seattle | P50145 | VNCCT8L24Z | 8C28043 |
| 4 | Columbia Hospitality | N/A | Seattle | P50145 | VNCCT8L251 | 8C28044 |
| 5 | Columbia Hospitality | N/A | Seattle | P50145 | VNCCT8L25J | 8C28045 |
| 6 | Spokane at Rainier Court | Joe Blow | Seattle | CB251I | ADXM013024710 | 2S31994 |

### Visual Design

Both pages share the same light theme. Background `#f8f9fa`, white card container (`border-radius: 12px`, `box-shadow: 0 2px 16px rgba(0,0,0,0.07)`), dark text `#2d3436`.

- Button colors: Paste `#6c5ce7`/white, Upload `#0984e3`/white, Clear `#e17055`/white, Copy `#00b894`/white, Copy & Open `#0984e3`/white, Copied feedback `#f39c12`/white.
- Nav links: border/color `#0984e3`, hover fills with `#0984e3`.
- Call cards: white bg, `#dee2e6` border, hover `#74b9ff` border with `rgba(116,185,255,0.15)` glow.
- Call badge: `#e9ecef` bg / `#495057` text. Field tiles: `#f8f9fa` bg.
- Status messages (inline, 5s auto-dismiss): success `#d4edda`/`#155724`, error `#f8d7da`/`#721c24`, info `#d1ecf1`/`#0c5460`.
- Responsive at ≤600px: single-column grid, stacked buttons, full-width action buttons.

---

## `call_notes.html` — Equipment Replacement Info: Full Specification

### Page Sections (top to bottom)

1. Title: "Equipment Replacement Info"
2. Two paste buttons + hint line
3. **Call Notes** (`h2`): Tech Name, City, Customer Name, Point of Contact, IT Person, Phone (`tel`), Email (`email`), Location of Machine, Model, New Equipment ID, Serial Number, Equipment ID Being Replaced
4. **IP Info** (`h3`): IP Method (`select`: Reused / Mac address reservation / DHCP / IP scanner / Other), + conditional "Specify Other" text input shown only when "Other" selected; IP Address, Subnet, Gateway, DNS 1–3
5. **Settings Transfer** (`h3`): textarea "How were settings transferred?"
6. **Scanning Info** (`h3`): Scan to Folder Type (`select`: SMB / WebDAV / FTP), Fax? (radio Yes/No)
7. **SMTP** (`h4`): Server, Port, SSL/TLS (`select`: None / SSL / TLS / STARTTLS), Email Address — **no password field**
8. **Reporting** (`h3`): four links (open in new tab), MPS? (radio Yes/No), Was manufacturer reporting enabled? (radio Yes/No)
9. **Page Counts** (`h4`): Black & White + Color number inputs side-by-side; live Total = B&W + Color with `toLocaleString()`, styled with `#f1f3f5` bg, bold blue `#0984e3` number
10. **Additional Info** (`h3`): hint "Fax details, user box, etc.", textarea
11. **Summary Output** (`h3`): read-only monospace textarea (`#f5f5f5` bg), also persisted in sessionStorage
12. Button group: Generate Summary (`#00b894`), Copy to Clipboard (`#0984e3`), Download Filled PDF (`#fdcb6e`/`#333`), Clear All (`#e17055`)

### Reporting Section Links

| Label | URL |
|---|---|
| MPS Monitor Portal ↗ | `https://portal.mpsmonitor.com/` |
| Jot Form ↗ | `https://form.jotform.com/250095273606153` |
| Vcare ↗ | `https://apps.kmbizhubvcare.com/vcaremobile/app/index.html#login?backUri=search` |
| Sharepoint ↗ | `https://pacificoa.sharepoint.com/sites/POALearningandDevelopment/SitePages/Remote-Monitoring-Tools.aspx` |

These links must **never** appear in the generated summary output.

### "Paste Calls" Button Logic

Reads clipboard, parses `Key: Value` lines, maps to fields (case-insensitive partial key match):

| Key contains | Maps to |
|---|---|
| "title" or "customer" | Customer Name |
| "contact" | Point of Contact |
| "item no" or "model" | Model |
| "serial" | Serial Number |
| "equipment id" or "equipment" | New Equipment ID |
| "city" | City |
| "technician" | Tech Name |

Auto-filled fields flash with `@keyframes flashHighlight` (1.5s fade from `#dee2e6` to transparent). Show toast with count.

### "Paste Summary" Button Logic

Re-imports a previously generated summary. Tracks current section by parsing header lines (`===`, `---`, `--`). Skips these fields (handled by Paste Calls or job-specific):

> Customer Name, Point of Contact, Model, Serial Number, New Equipment ID, Equipment ID Being Replaced, Location of Machine, IP Address, Black & White, Color, Total

**Email disambiguation**: if current section is "smtp", map "Email" → `smtpEmail`; otherwise → `itEmail`.

**IP Method**: match against known options; if value starts with `"Other:"` set dropdown to Other and put remainder in conditional field; trigger `toggleOtherIP()` after setting.

### Generate Summary Format

Only include a section header if at least one field in it has data. Only include lines with data.

```
=== CALL NOTES ===
Customer Name: {value}
Point of Contact: {value}
IT Person: {value}
Phone: {value}
Email: {value}
Location of Machine: {value}
Model: {value}
New Equipment ID: {value}
Serial Number: {value}
Equipment ID Being Replaced: {value}

--- IP Info ---
IP Method: {value or "Other: custom text"}
IP Address: {value}
Subnet: {value}
Gateway: {value}
DNS 1: {value}
DNS 2: {value}
DNS 3: {value}

--- Settings Transfer ---
How settings were transferred: {value}

--- Scanning Info ---
Scan to Folder Type: {value}
Fax: {Yes/No}

-- SMTP --
Server: {value}
Port: {value}
SSL/TLS: {value}
Email: {value}

--- Reporting ---
MPS: {Yes/No}
Manufacturer reporting enabled: {Yes/No}
-- Page Counts --
Black & White: {number with commas}
Color: {number with commas}
Total: {number with commas}

--- Additional Info ---
{free text}
```

### PDF Field Mappings (`net_form_with_text_feilds.pdf`)

Note: filename misspelling is intentional.

| PDF field name | Web form field |
|---|---|
| `customer` | Customer Name |
| `equipID` | New Equipment ID |
| `model` | Model |
| `black` | Black & White count |
| `color` | Color count |
| `total` | B&W + Color as string |
| `ip` | IP Address |
| `subnet` | Subnet |
| `gateway` | Gateway |
| `dns1` | DNS 1 |
| `dns2` | DNS 2 |
| `city` | City |
| `techName` | Tech Name |
| `location` | Location of Machine |

Use `form.getTextField(name).setText(value)`. Wrap each in `try/catch`. Do **not** flatten. Download as `Network_Install_{CustomerName}_{EquipID}.pdf` (sanitize non-alphanumeric to `_`; default "Unknown"/"NoID" if blank).

### Styling

- System font stack. Background `#f8f9fa`. Container: white card, `max-width: 640px`, `border-radius: 12px`, `box-shadow: 0 2px 16px rgba(0,0,0,0.07)`, padding `32px 28px`.
- Inputs: `1px solid #dee2e6`, `8px border-radius`, focus ring `#74b9ff` with `0 0 0 3px rgba(116,185,255,0.35)`. All selects: `-webkit-appearance: none`. Radio buttons: `-webkit-appearance: radio; appearance: radio`.
- `.hidden`: `display: none !important`.
- Mobile breakpoint `480px`: reduce padding, stack button rows and page count fields vertically, paste buttons full width.
