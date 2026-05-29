# Tracker Dev Notes
_Last updated: 2026-05-29_

---

## Architecture

Single HTML file · localStorage · Deployed to GitHub Pages  
Live URL: https://t7pn725hwt-netizen.github.io/Nutrition-Tracker/  
No build step, no dependencies, no backend.

The entire app — UI, logic, data layer, and report builder — lives in `index.html`.

---

## localStorage Keys

All keys currently used by the app:

| Key | Contents |
|-----|----------|
| `cfTrackerData_v1` | Main data blob: meals (by date), favorites, goals |
| `coimasDoseSettings_v1` | Insulin settings: ICR, glucose target, ISF by time of day |
| `cfTrackerSettings_v1` | App settings: Anthropic API key, GitHub credentials, last sync time |
| `cfLearningCorrections_v1` | AI photo estimate correction history (last 50 corrections) |
| `glucose:YYYY-MM-DD` | Raw glucose samples array for that date (from HealthKit Shortcut) |
| `report:YYYY-MM-DD` | Computed daily report JSON (pre-built before sending to Claude) |
| `rlib_YYYY-MM-DD` | Saved daily report HTML (imported from Claude artifact) |
| `rlib_w_YYYY-MM-DD` | Saved weekly report HTML (imported from Claude artifact) |
| `rlib_meta_YYYY-MM-DD` | Extracted metadata for a daily report (TIR, protein, avg glucose) |
| `rlib_w_meta_YYYY-MM-DD` | Extracted metadata for a weekly report |
| `rlib_index` | JSON array of saved daily report dates, sorted newest first |
| `rlib_w_index` | JSON array of saved weekly report dates, sorted newest first |

**Note for iOS conversion:** The Anthropic API key and GitHub PAT are currently stored in `cfTrackerSettings_v1` via plain localStorage. These must move to iOS Keychain before App Store submission — storing credentials in key-value storage is not permitted.

---

## External Dependencies

**Cloudflare Workers proxy** (`https://little-fog-50ac.d22f8kg97r.workers.dev/`)  
Hardcoded in three places in the JS (all marked with comments):
- Anthropic API calls (`/anthropic/v1/messages`) — proxy adds CORS headers required for browser→API calls
- OpenFoodFacts barcode lookups (`/off/api/v2/product/…`) — proxy rewrites to openfoodfacts.org
- Favorite photo match calls (also Anthropic)

In a native iOS app, all three can be called directly — no proxy needed. The Anthropic iOS SDK handles auth natively, and openfoodfacts.org has no CORS restrictions for native clients.

**Google Fonts** (`fonts.googleapis.com`) — Inter + Space Mono. Loaded at startup. Falls back gracefully if offline.

---

## Insulin Logic

- **Humalog**: meal dose only. Calculated from net carbs ÷ ICR, plus BG correction.
- **Lantus**: basal only. Logged as a meal entry at dose time. Never factors into IOB or ICR calculations — filtered by name check (`includes('lantus')`).
- **IOB**: computed across all active Humalog doses in the past 4 hours using a linear decay model (full decay at 4 hours). Stacked doses are summed into a single aggregate IOB before each calculation. See `getActiveIob()`.
- **Pre-bolus**: timing recommendation is BG-aware — shifts to first bite or with food when BG is running low. Logged in `doseLog.preBolus`.
- **Corrections**: standalone Humalog entries. The dose calculator handles correction-only mode when carbs = 0.
- **Dose settings**: ICR, glucose target, and ISF (morning/afternoon/evening) are user-configurable in Settings. All calculator logic reads from `getDoseSettings()` — no hardcoded constants anywhere.

---

## ICR Calibration — Exclusion Flags

A meal is `icr_calibration.eligible: false` if any of these are true:

| Flag | Condition |
|------|-----------|
| `high_fat` | meal fat_g > 40 |
| `otf_day` | any workout recorded that day |
| `pre_bg_elevated` | pre-dose BG > 140 |
| `correction_given` | correction dose within 4 hours of meal |
| `stacking_risk` | previous Humalog dose within 3 hours |
| `response_unresolved` | glucose not back within 20% of pre-dose BG within 4 hours |

High-fat meals excluded to keep ICR data clean.  
OTF days excluded due to post-workout insulin sensitivity masking true ICR.

---

## HealthKit Glucose Pull Windows (CDT / UTC-5)

Four isolated calls, no UTC midnight crossings. Fixed anchors — never derive from sample timestamps.

| Window | UTC Start | UTC End | CDT Equivalent | Limit |
|--------|-----------|---------|----------------|-------|
| A | `YYYY-MM-DDT05:00:00Z` | `YYYY-MM-DDT20:00:00Z` | midnight → 3pm | 220 |
| B1a | `YYYY-MM-DDT20:00:00Z` | `YYYY-MM-(DD+1)T00:00:00Z` | 3pm → 7pm | 70 |
| B1b | `YYYY-MM-(DD+1)T00:00:00Z` | `YYYY-MM-(DD+1)T05:00:00Z` | 7pm → 11pm | 70 |
| B2 | `YYYY-MM-(DD+1)T05:00:00Z` | `YYYY-MM-(DD+1)T11:00:00Z` | 11pm → 5am | 90 |

If any window returns `samples_truncated: true`:  
Gap fill → start: last timestamp of preceding window, end: oldest timestamp of truncated window, limit 20.

CST (Nov–Mar, UTC-6): shift all anchors one hour later. The app auto-detects CDT vs CST via `chicagoOffsetForDate()`.

---

## Glucose Pipeline (Current Flow)

1. User runs the **Glucose Pull** iOS Shortcut — copies newline-separated JSON to clipboard
2. User taps **📋 Import** in the tracker — `parseGlucoseClipboard()` normalizes the data
3. Samples stored under `glucose:YYYY-MM-DD` in localStorage
4. When **⬡ Daily Report** is tapped, `buildReportJSON()` computes TIR, per-meal glucose response, and ICR calibration — stores as `report:YYYY-MM-DD`
5. The full JSON is copied to clipboard as a structured prompt for Claude
6. Claude builds the report HTML artifact; user pastes it into the Reports tab via **+**

The Shortcut is documented in `SHORTCUT-SETUP.md`. For the iOS native app, HealthKit can be queried directly — the Shortcut step is not needed.

---

## Delta Compression for Chart Samples

Store a glucose point only if:
- `|current_value - last_stored_value| >= 3 mg/dL`, OR
- `>= 15 minutes elapsed since last stored point`

Always store first and last sample of each window.  
Target: 60–90 points per day. Clinically sufficient for trend analysis and peak/nadir detection.

---

## TIR Computation

- **Source**: CDT 00:00–23:59 samples only
- Post-midnight data goes to chart arrays but not TIR counts
- Sensor warmup gap (~2 hours every 15 days) expected — flag but don't penalize TIR
- Any other gap > 20 minutes: flag as pull error

---

## Report JSON

Daily report stored as `report:YYYY-MM-DD` in localStorage.  
Weekly rollup aggregates 7 daily objects — no re-pulling required.  
`week_ref` format: `YYYY-WNN` (ISO week number).

Key additions over raw meal log: `glucose_response` and `icr_calibration` blocks per meal.

---

## Report Storage (rlib_*)

Reports are saved as raw HTML strings (the Claude artifact output).  
Each report has a companion metadata record (`rlib_meta_*`) with extracted values for the index view (TIR, protein, avg glucose).

The Reports UI is a full-screen overlay rendered via a sandboxed `<iframe>` with `srcdoc`. An injected script uses `postMessage` to report scroll height back to the parent for auto-resizing.

**For iOS:** Replace the iframe with `WKWebView` using `loadHTMLString(_:baseURL:)`. The auto-resize postMessage mechanism can stay as-is — WKWebView supports it natively.

---

## GitHub Auto-Sync

Meal data and saved reports sync automatically to a private GitHub repo (`nutrition-tracker`) after each save, debounced at 30 seconds. Credentials stored in `cfTrackerSettings_v1`. Entry point: `githubSync()`. Manual trigger available in Settings.

**For iOS:** GitHub sync can stay or be replaced with iCloud sync — either is straightforward. Credentials must move to Keychain regardless.

---

## Dexcom Direct Integration (Planned)

- Apply for sandbox access: developer.dexcom.com
- OAuth2 flow — user authenticates with Dexcom account
- Replaces HealthKit Shortcut pull entirely
- Real-time streaming vs 5-min HealthKit sync
- Status: **not started**

---

## Web APIs That Need Native Replacements (iOS)

| Web API | Used For | iOS Replacement |
|---------|----------|-----------------|
| `BarcodeDetector` | Barcode scanning from photo | AVFoundation / Vision framework |
| `<input type="file" accept="image/*">` | Camera / photo library access | `PHPickerViewController` or `UIImagePickerController` |
| `navigator.clipboard.writeText()` | Copy report prompts to clipboard | `UIPasteboard` |
| `<iframe srcdoc>` | Render saved report HTML | `WKWebView` + `loadHTMLString` |
| `localStorage` | All data storage | `UserDefaults` (data) + Keychain (credentials) |

---

## Design System

```
--green-1: #4dbb88   in-range glucose, goal hit
--blue:    #3a8fd4   insulin, confirmed values
--orange:  #d4954a   above 180, warnings
--red:     #d44a4a   below 70, missed goals
--purple:  #9d6fff   snack meal pills

Background: #0b0d0f
Surface:    #13161a
Border:     #1f242a

Fonts: Space Mono (numbers, labels, mono) · Inter (body)
```

---

## Known Issues / Open Items

- Activity toggle (light activity after eating) logs for future data only — does not currently affect dose calculation
- OTF workout data sometimes delayed in HealthKit pull (confirmed May 2026 — likely sync delay at pull time)
- Dexcom direct integration not started — currently uses HealthKit Shortcut as glucose source
