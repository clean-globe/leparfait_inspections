# Le Parfait

A lightweight web app for coordinating cleaning-site inspections across states and supervisors. Supervisors record which sites they inspect and on what schedule; changes are written straight to a Google Sheet through a Google Apps Script webhook — no database, no hosted backend.

**Live URL:** https://leparfait.vercel.app/

---

## Overview

The app is a two-panel dashboard with a persistent sidebar (top-bar hamburger drawer on mobile) that routes between three views:

- **Site Details** — placeholder
- **Site Inspection Details** — the functional supervisor-facing form, reads and writes the Google Sheet
- **Test and Tags** — placeholder

The system is intentionally simple — one HTML file, one Apps Script file, one Google Sheet — so it can be maintained without a build step, a framework, or any hosted infrastructure beyond static file hosting.

---

## Architecture

```
┌──────────────────────┐        POST/GET         ┌──────────────────────┐
│   Vercel             │  ────────────────────>  │  Google Apps Script  │
│   (static hosting)   │       JSON payload      │   (doPost / doGet)   │
│                      │                         │                      │
│   index.html         │  <────────────────────  │                      │
└──────────────────────┘        JSON reply       └──────────┬───────────┘
                                                            │
                                                            │ reads/writes
                                                            ▼
                                                 ┌──────────────────────┐
                                                 │  Google Sheet        │
                                                 └──────────────────────┘
```

- **Frontend:** single-page HTML/CSS/JS on Vercel, auto-deployed from GitHub on push to `main`
- **Backend:** Google Apps Script web app bound to the sheet, deployed as **Anyone can access**
- **Storage:** a single Google Sheet — the first sheet of the spreadsheet bound to the Apps Script

---

## Layout

- **Desktop (≥ 768px):** fixed 240px sidebar on the left, main content on the right
- **Mobile (< 768px):** sidebar slides off-screen; a ☰ hamburger in the top bar opens a drawer with a dark overlay. Tapping the overlay or picking a nav item closes it automatically
- **Top bar** is sticky and shows the brand "Le Parfait" on all screen sizes

---

## Views

### 1. Site Details
Placeholder card ("🚧 This section is under construction. UI to be defined.").

### 2. Site Inspection Details

The main functional view — where supervisors register their inspection commitments.

**Flow:**
1. User picks **State → Supervisor** (cascading dropdowns)
2. On supervisor select, the frontend fetches that supervisor's most recent submission and pre-populates the **Sites to be Inspected** table
3. User clicks **Add New Site Inspection Details** to open the form
4. Fills in Site Name and Inspection Schedule → clicks **Add Site to List** → the site is written to the spreadsheet immediately
5. Edit or Remove on existing rows also saves immediately

**Auto-save flow:** every add / edit / remove triggers a POST to the Apps Script that rewrites the supervisor's site list. No batch "Save & Submit" step — changes are persisted the moment the user commits them in the form. A brief green banner ("✓ Site added / updated / removed") appears after each save; a red banner shows if the network request fails, in which case local state rolls back to match the sheet.

**Sites to be Inspected table:**
- Columns: `#`, `Site Name`, `Schedule`, and a narrow actions column
- **Schedule column** renders grid entries as one line per cell (e.g. `Week 1 - Mon` / `Week 2 - Mon` on separate lines). Custom entries render as-is
- **Actions column** shows a highlighted square chevron button. Clicking it slides in a floating panel with **Edit** and **Remove**. The chevron rotates 180° and turns solid blue when open. Only one row can be open at a time; clicking anywhere else closes it

**Add Site Inspection Details form** (hidden until the Add button is pressed):
- **Site Name** — free text
- **Inspection Schedule** — one of two modes chosen via radio, separated by an `OR` divider:
  - **📅 Use 4-week grid** — click cells in a 4×7 grid (Week 1–4 × Mon–Sun) representing a repeating 4-week cycle. Clicking a day header (Mon, Tue…) toggles that whole column across all four weeks — the quick way to set a "weekly" pattern
  - **✏️ Custom frequency (Other)** — free-text field for schedules that don't fit the grid (e.g. `Every 3 months`, `Ad-hoc`, `Twice a year`)
- **+ Add Site to List** button saves immediately; **Cancel** hides the form without saving
- When editing an existing row, the form pre-fills with its values and the button label changes to "Save Changes"

### 3. Test and Tags
Placeholder card ("🚧 This section is under construction. UI to be defined.").

---

## Data Model

### Google Sheet

| Column | Field          | Example                                    |
|--------|----------------|--------------------------------------------|
| A      | Timestamp      | `2025-11-18 14:32:07`                      |
| B      | State          | `NSW`                                      |
| C      | Supervisor     | `Yomal`                                    |
| D      | Site Name      | `Westfield Bondi`                          |
| E      | Schedule       | `Week 1 - Mon, Week 3 - Mon` OR `Ad-hoc`   |
| F      | Submission ID  | `550e8400-e29b-41d4-a716-446655440000`     |

- **One row per site.** A supervisor with 5 sites produces 5 rows sharing the same Submission ID
- **Delete-and-replace, not append-only.** Each save deletes all existing rows for that state+supervisor and inserts the new set. The sheet only ever holds the current state
- **Schedule format is uniform.** Grid entries: `Week N - Day, Week N - Day, …`. Custom entries: whatever the user typed

### Wire Format (frontend ↔ Apps Script)

**POST payload (frontend → Apps Script):** sent on every add / edit / remove
```json
{
  "state": "NSW",
  "supervisor": "Yomal",
  "sites": [
    { "siteName": "Westfield Bondi", "schedule": "Week 1 - Mon, Week 3 - Mon" },
    { "siteName": "Central Plaza",    "schedule": "Every 3 months" }
  ],
  "submittedAt": "2025-11-18T14:32:07.123Z"
}
```

**GET response (Apps Script → frontend):** used to pre-populate the table on supervisor select
```json
{
  "success": true,
  "sites": [
    { "siteName": "Westfield Bondi", "schedule": "Week 1 - Mon, Week 3 - Mon" },
    { "siteName": "Central Plaza",    "schedule": "Every 3 months" }
  ],
  "submittedAt": "2025-11-18T14:32:07.123Z"
}
```

On edit, the frontend parses the schedule string: if every comma-separated part matches `Week N - Day`, it opens the 4-week grid preselected with those cells; otherwise it opens custom-frequency mode with the raw text.

---

## API

The Apps Script web app exposes two endpoints at the same URL.

### `POST /exec`
Body: JSON payload as above.
- Serializes concurrent submissions via `LockService.getScriptLock()` (30-second wait)
- Deletes all existing rows matching `state + supervisor`
- Inserts one row per site with a new shared Submission ID
- Calls `SpreadsheetApp.flush()` to commit before releasing the lock
- **Note:** the frontend uses `mode: "no-cors"` to avoid CORS preflight, so it cannot read the response — it treats "no thrown exception" as success

### `GET /exec?state=X&supervisor=Y`
- Returns the most recent submission for that state+supervisor
- Groups rows by Submission ID, picks the group with the latest timestamp
- Returns `sites: []` if no matching data exists

---

## File Structure

```
project/
├── index.html      # Frontend — HTML, CSS, and JS in a single file
├── Code.gs         # Apps Script backend (paste into script.google.com)
└── README.md       # This file
```

The frontend is self-contained. No build step, no dependencies, no framework.

---

## Adding a New View

The sidebar and view system is designed to make adding views trivial. Three changes to `index.html`:

**1.** Add a sidebar nav item:
```html
<a class="nav-item" data-view="myNewView">
  <span class="nav-icon">📊</span>
  <span>My New View</span>
</a>
```

**2.** Add a view container inside `<main class="main">`:
```html
<div class="view" id="view-myNewView">
  <div class="container">
    ... your content ...
  </div>
</div>
```

**3.** No JS change needed — the existing routing wires `data-view="myNewView"` to `id="view-myNewView"` automatically.

The nav item's `data-view` attribute must match the view's `id` (minus the `view-` prefix).

---

## Setup / Deployment

### 1. Create the Google Sheet

Create a new Google Sheet with these headers in row 1 (order matters):

| Timestamp | State | Supervisor | Site Name | Schedule | Submission ID |
|-----------|-------|------------|-----------|----------|---------------|

### 2. Deploy the Apps Script

1. From the sheet: **Extensions → Apps Script**
2. Delete the boilerplate and paste `Code.gs`
3. Save
4. **Deploy → New deployment → Web app**
   - Execute as: **Me**
   - Who has access: **Anyone**
5. Authorize when prompted
6. Copy the **Web app URL**

### 3. Configure the frontend

In `index.html`, near the top of the `<script>` block, set `WEBHOOK_URL`:

```javascript
const WEBHOOK_URL = "https://script.google.com/macros/s/YOUR_DEPLOYMENT_ID/exec";
```

### 4. Deploy to Vercel

1. Push the repo to GitHub
2. In Vercel: **New Project** → import the repo
3. Framework preset: **Other** (no build step)
4. Deploy

Any push to `main` triggers a production redeploy. Pushes to any other branch create a **preview deployment** at `leparfait-git-<branch-name>-<username>.vercel.app` while production stays on the last `main` commit — useful for testing changes without touching the live URL.

### 5. Customize supervisors

Edit the `supervisorsByState` map in `index.html`:

```javascript
const supervisorsByState = {
  WA: ["Janith", "Madushanka", "MJ"],
  ACT: ["Shelly"],
  QLD: ["Maggie", "Ranveer"],
  NSW: ["Yomal", "Charith", "Ikram", "Uma", "Hassan"],
  VIC: ["Dinesh", "Anish"]
};
```

Push to GitHub — Vercel auto-deploys.

---

## Updating the Deployed Apps Script

When you change `Code.gs`, **saving is not enough** — the live web app runs whatever version you deployed. To push new code:

1. In the Apps Script editor: **Deploy → Manage deployments**
2. Click the **pencil icon** on your existing deployment
3. Version dropdown → **New version**
4. **Deploy**

The URL stays the same, so the frontend keeps working with no changes.

**Do not click "New deployment"** — that creates a separate URL and breaks the connection to your frontend unless you also update `WEBHOOK_URL`.

---

## Known Trade-offs and Design Decisions

### Auto-save with no-cors POST
Every add / edit / remove triggers a POST. Because `mode: "no-cors"` (needed to avoid an OPTIONS preflight that Apps Script can't answer), the frontend cannot read the server's response body — it treats "no thrown exception" as success. That means:
- **Network failures** (offline, DNS, timeout) are caught and rolled back
- **Server-side failures** (Apps Script exception, quota, sheet locked) are *not* caught and will silently show "Saved" on the client while the sheet stays unchanged

### Delete-and-replace, not append-only
Each save replaces the supervisor's previous rows. The sheet stays clean and matches the mental model ("this is my current site list"), but there's no submission history.

### Concurrency
`LockService.getScriptLock()` serializes POST handlers so simultaneous submissions don't corrupt each other. Without this, two supervisors submitting at the same moment (or one supervisor open in two tabs) could see stale row indices between the delete and insert steps and clobber each other's data. The 30-second wait window is generous — each POST completes well under a second under normal load.

### Site Details and Test and Tags are placeholders
Both views currently render a "🚧 Under construction" card and have no wiring or data source.

### No authentication
Any visitor with the URL can submit as any supervisor. Fine for a small internal tool, but if the form is exposed publicly, someone could write garbage rows against arbitrary supervisor names.
