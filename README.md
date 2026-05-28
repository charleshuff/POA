# Link to Call Parser
https://poa.charleshuff.net

## List of Driver Websites
[Konica Print Drivers](https://onyxweb.mykonicaminolta.com/OneStopProductSupport?appMode=public&productId=2175&categoryId=1&subCategoryId=ft0)

[Sharp](https://business.sharpusa.com/Product-Downloads)

[Lexmark](https://support.lexmark.com/en_us/drivers-downloads.html) 

[HP](https://support.hp.com/us-en/drivers/printers)

[Canon](https://www.usa.canon.com/support/software-and-drivers)

[Ricoh](https://www.ricoh-usa.com/en/support-and-download)

# POA Field Service Tools

A set of client-side web tools for Pacific Office Automation field service technicians. No backend, no build step — just open in a browser or deploy via GitHub Pages.

---

## Call List Parser (`index.html`)

Parses raw HTML copied from the ETMS web portal into structured call cards.

### How to use

1. Log into ETMS (`etms.pacificoffice.com`) and copy your call list HTML (Ctrl+U,Ctrl+A, Ctrl+C on the page, or right click and select view source).
2. Open the Call List Parser and click **Paste**.
3. The app parses the HTML and displays a card for each call with: Customer, Contact, City, Model, Serial Number, Equipment ID, and Technician.
4. Click **Copy & Open Call Notes** to copy the details and jump straight to the Call Notes page.

### What it extracts

- **Technician name** from the ETMS settings panel (shared across all calls).
- **Per-call fields:** Customer, Contact, City, Item No. (Model), Serial Number, Equipment ID.

---

## Equipment Replacement Info (`call_notes.html`)

A form for documenting an equipment replacement job. Generates a text summary and a filled PDF.

### How to use

1. Arrive from the Call List Parser (or open directly).
2. Click **Paste Calls** to auto-fill fields from the clipboard (copied by the parser). Fields flash to confirm they were filled.
3. Fill in the remaining sections:
   - **Call Notes** — Customer, contacts, location, model/serial/equipment IDs.
   - **IP Info** — IP method, address, subnet, gateway, DNS servers.
   - **Settings Transfer** — How settings were moved to the new machine.
   - **Scanning Info** — Folder type (SMB/WebDAV/FTP), fax status.
   - **SMTP** — Mail server, port, SSL/TLS, email address.
   - **Reporting** — MPS status, manufacturer reporting. Links to MPS Monitor Portal, Jot Form, Vcare, and Sharepoint.
   - **Page Counts** — Black & white and color counts (total calculated automatically).
   - **Additional Info** — Free-text notes.
4. Click **Generate Summary** to produce a formatted text summary and copy it to clipboard.
5. Click **Copy to Clipboard** to copy the summary.
6. Click **Download Filled PDF** to get a pre-filled network install form.

### Re-importing a summary

Click **Paste Summary** to re-import a previously generated summary back into the form fields. Only information that doesn't change from machine to machine will be imported. 

### PDF output

Downloads as `Network_Install_{Customer}_{EquipID}.pdf` with IP info, page counts, and other key fields filled in.

---

## Screenshot Keeper (`screenshot_keeper.html`)

A clipboard-based screenshot gallery. Paste screenshots, name them, and download them later.

### How to use

1. Take a screenshot (or copy any image to your clipboard).
2. Open Screenshot Keeper and press **Ctrl+V** (or **Cmd+V** on Mac) anywhere on the page.
3. A modal appears with a preview — type a name and press **Enter** to save.
4. The image appears in the gallery as a card with a thumbnail.

### Features

- **Click a thumbnail** to open it in a full-screen lightbox.
- **Download** a single image or use **Download All** to grab everything.
- **Delete** individual images or **Clear All** to reset the gallery.
- All images are stored in `sessionStorage` — they persist across page reloads but are cleared when you close the tab.

---

## General Notes

- All pages are standalone HTML files with no external dependencies (except `call_notes.html` which loads `pdf-lib` from CDN for PDF generation).
- All form fields auto-save to `sessionStorage` on every keystroke, so nothing is lost on accidental refresh.
- Pages communicate via the clipboard only — there is no shared storage between them.
- Works on mobile with responsive layouts.
