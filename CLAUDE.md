# Evans Property Management — Project Reference

## What This Is
A two-page web app for managing holiday let changeovers in West Wales.

- **index.html** — Cleaner-facing app. Cleaners log in by name, see today's changeovers and a 14-day schedule, select a property, pick the booking, tick extras used, log hours/notes, and submit.
- **dashboard.html** — Admin dashboard (PIN protected, PIN: `1234`). Shows reported issues, performance stats, and generates invoices per property per month.

Both are static HTML files deployed on Vercel. No build step, no framework — plain HTML/CSS/JS.

---

## Architecture

```
Vercel (static hosting)
  ├── index.html        (cleaner app)
  └── dashboard.html    (admin dashboard)
        ↓ fetch GET/POST
Google Apps Script (web app endpoint)
        ↓ read/write
Google Sheets (data store)
  ├── Bookings tab      (pulled from booking system)
  └── Changeovers tab   (written by cleaner app submissions)
```

---

## Google Apps Script

**Live endpoint URL (deployed 4 May 2026):**
```
https://script.google.com/macros/s/AKfycby6ude4tWpwxI2MChF8qdeNOptOm3lZDdDdVTWcrE_4FTgfGItQx9qKEIlIxCEuoTxq6Q/exec
```

**Previous URL (retired — was pointing to old project):**
```
https://script.google.com/macros/s/AKfycby0Ew_Cx7283p5G7Xlcd4JuDus0llbmT9bXB2ldGeeVFR6gjbJmlbnAGrKzkckE0RCIEQ/exec
```

**Old URL (retired):**
```
https://script.google.com/macros/s/AKfycbxUDsON3GhWBoqlTaChyHCWKHutuvT4RFiZpbEUi5dG_-6yMM-GpsGUoMwQP7x922lnYA/exec
```

**Google Sheet ID:** `12UWt82DkqB2kF1tUm2ekB9rt82gsnhciPgJHV7L8_SA`

**Sheet tab names:**
- `Bookings` — incoming bookings from the booking system
- `Changeovers` — logged changeover records written by the cleaner app

**Apps Script actions:**
- `GET ?action=bookings` — returns JSON array of upcoming bookings (yesterday → +60 days)
- `GET ?action=changeovers` — returns JSON array of all logged changeovers (newest first)
- `POST` with `data=JSON` — writes a new changeover row to the Changeovers sheet

**Deployment settings:**
- Execute as: **Me**
- Who has access: **Anyone** (no Google account required — critical for CORS)
- Each time you edit the script, create a **New Version** in Manage Deployments — do not create a brand new deployment or the URL will change.

---

## Properties

Defined in `PROPERTIES` array in `index.html` and `PROPS_CFG` / `OWNER_DATA` in `dashboard.html`.

| Property | ID | Ref | Rate Type | Short | Long | Threshold | Flat | Mgmt Fee |
|---|---|---|---|---|---|---|---|---|
| Maes Dyfed | maes-dyfed | MD | split | £130 | £170 | 4 nights | — | £80 |
| Arosfa | arosfa | A | split | £130 | £170 | 4 nights | — | £99 |
| Teifi View | teifi-view | TV | split | £110 | £130 | 4 nights | — | — |
| Ty Gladys | ty-gladys | TG | split | £125 | £160 | 4 nights | — | £79 |
| Ty Eufron | ty-eufron | TE | split | £150 | £180 | 4 nights | — | £79 |
| Brynhowell | brynhowell | BFH | flat | — | — | — | £250 | — |
| Stambar | stambar | STB | flat | — | — | — | £140 | — |
| Ty Pinc | ty-pinc | TP | flat | — | — | — | £130 | — |
| Cwm Hyfryd | cwm-hyfryd | CH | all-inc | — | — | — | £260 | — |

**Rate types:**
- `split` — short stay vs long stay rate, split at `threshold` nights
- `flat` — single flat rate regardless of nights
- `allinc` — all-inclusive (linen, bins, welcome pack included in rate)

**To add/remove a property:** Update `PROPERTIES` in `index.html` AND `PROPS_CFG` + `OWNER_DATA` in `dashboard.html`. These are duplicated — keep them in sync. *(Future improvement: extract to a shared config.js)*

---

## Known Bugs Fixed (May 2026)

### 1. Dashboard showed no data
**File:** `dashboard.html` line 312–313
**Cause:** `loadData()` assumed the API returned `{ data: [...] }` but it returns a raw array.
```javascript
// WRONG (old):
changeovers = cd.data || [];
bookings    = bd.data || [];

// FIXED:
changeovers = Array.isArray(cd) ? cd : (cd.data || []);
bookings    = Array.isArray(bd) ? bd : (bd.data || []);
```

### 2. Schedule showed no upcoming changeovers
**Cause:** The original Apps Script only returned bookings with checkout dates up to today. The 14-day schedule strip needs future bookings.
**Fix:** New Apps Script deployed with a window of yesterday → +60 days.

---

## Apps Script — Canonical Version
The working script (deployed May 2026) uses `var` throughout (not `const` at top level — Apps Script Rhino runtime doesn't support it). Key logic:

```javascript
// Booking window — the critical filter
var today = new Date(); today.setHours(0, 0, 0, 0);
var windowStart = new Date(today); windowStart.setDate(today.getDate() - 1);
var windowEnd   = new Date(today); windowEnd.setDate(today.getDate() + 60);
```

Full script is stored separately. The sheet is opened by ID inside each function:
```javascript
var ss = SpreadsheetApp.openById('12UWt82DkqB2kF1tUm2ekB9rt82gsnhciPgJHV7L8_SA');
```

---

## Vercel Deployment
- Project lives at: https://vercel.com/evans-projects-eac0a5db
- Deployed from a private GitHub repo
- No build step — Vercel serves the HTML files directly

---

## Booking Automation (fetch-bookings/)

A Node.js + Playwright script that logs into Awaze and Classic portals, intercepts their API calls, and pushes all bookings to the Google Sheet automatically.

**Location:** `fetch-bookings/fetch-bookings.js`

**Credentials:** `fetch-bookings/.env` (never commit this file)

**Runs:** Daily at 6am via cron job (`crontab -l` to view)

**Logs:** `/tmp/epm-bookings.log`

**To run manually:**
```bash
cd "/Users/evandavies/Desktop/Evans Property Management/fetch-bookings"
node fetch-bookings.js
```

**How it works:**
1. Launches headless Chromium via Playwright
2. Logs into `owners.awaze.com` with credentials from `.env`
3. Logs into `portal.classic.co.uk` with credentials from `.env`
4. Intercepts JSON API responses containing booking data
5. Normalises property names via `PROPERTY_MAP` in the script
6. Deduplicates across both platforms
7. POSTs to Apps Script `?action=updateBookings` which replaces the Bookings sheet

**Apps Script changes needed (in APPS_SCRIPT_UPDATE.gs):**
- Replace `doPost` function with the version in `fetch-bookings/APPS_SCRIPT_UPDATE.gs`
- Add the `updateBookingsSheet()` function from the same file
- Secret key: `EPM2026`

**If bookings aren't showing after a run:**
- Check `/tmp/epm-bookings.log` for errors
- The script logs every API URL it captures — if 0 are captured, login failed
- To debug visually: change `headless: true` to `headless: false` in `fetch-bookings.js` to watch the browser

**Property name mapping:**
Edit the `PROPERTY_MAP` object in `fetch-bookings.js` if Awaze/Classic use different names than the app. The keys are lowercase versions of what the portal returns.

---

## Robustness Improvements (To Do)
1. **Shared config file** — extract `PROPERTIES`, `PROPS_CFG`, `OWNER_DATA` into a single `config.js` included by both HTML files. Currently duplicated with different field names (`rateFlat` vs `flatRate`).
2. **Apps Script health endpoint** — add `?action=health` returning `{ bookings: N, futureBookings: N, changeovers: N }` for quick diagnosis.
3. **GitHub repo** — `https://github.com/edavies775-ship-it/evanspropertymanagement` — both HTML files are version controlled, Vercel auto-deploys on push to `main`.
4. **Defensive API wrapper** — log a console warning if the API returns 0 records (catches silent failures early).

---

## How to Diagnose Booking Issues
1. Test the live endpoint directly in a browser or with curl:
   ```
   curl -L "https://YOUR_SCRIPT_URL/exec?action=bookings"
   ```
   Should return a JSON array. If it returns HTML, the script has an error or auth issue.

2. Check the sync status indicator in the cleaner app — red dot = API failed, gold = stale cache, green = live.

3. Check browser DevTools → Network tab → look for the Apps Script request and inspect the response.

---

## Google Account Notes
- The Google Sheet and original Apps Script are on a different Google account to the one used for incognito access.
- If you get an `authuser=2` error opening Apps Script, open in an incognito window logged in as the sheet-owner account only.
- The new standalone Apps Script (May 2026) uses `openById` so it can be owned by any account that has access to the sheet.
