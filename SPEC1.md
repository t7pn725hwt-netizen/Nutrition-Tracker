# Nutrition Tracker — Technical Specification

This document describes the architecture, data model, and UI of the Nutrition Tracker.  
**Read before making modifications.**

> **Deployment:** GitHub Pages (`https://t7pn725hwt-netizen.github.io/Nutrition-Tracker/`). Ignore older references to Netlify.

---

## Overview

A single self-contained HTML file (no backend, no build step) running entirely in the browser. All data stored in `localStorage`. Designed for iPhone home-screen use via Safari "Add to Home Screen".

Two surfaces work together:
1. **The tracker app** — logs meals, calculates insulin doses, generates report prompts
2. **A Claude project** — receives the report prompt, pulls blood glucose data directly from Apple HealthKit, builds the HTML report artifact, which is then pasted back into the tracker for storage

The tracker also calls Anthropic and OpenFoodFacts APIs directly (via a Cloudflare Workers proxy) for AI photo macro estimation and barcode lookup.

---

## File Structure

Single HTML file:
- `<style>` — all CSS and CSS custom properties (design tokens), no external CSS
- `<body>` — all HTML markup
- `<script>` — all JavaScript, no external JS

External dependencies:
- Google Fonts (`Inter` body, `Space Mono` numeric/header)
- Cloudflare Workers proxy at `https://little-fog-50ac.d22f8kg97r.workers.dev/` (CORS shim for Anthropic API + OpenFoodFacts)

---

## Design System

### Colors (CSS custom properties)
```css
--bg: #0b0d0f          /* page background */
--surface: #13161a     /* card background */
--surface-2: #181c21   /* slightly lighter surface */
--border: #1f242a      /* card borders */
--text: #e6e8eb        /* primary text */
--text-dim: #8b9199    /* secondary text, labels */
--text-faint: #555c66  /* de-emphasized text, units */
--green-1: #4dbb88     /* primary accent */
--green-2: #3a9970     /* mid green — section labels */
--green-3: #2a7055     /* deep green — button borders */
--green-4: #0f2a1e     /* darkest green — progress bar track */
--blue: #3a8fd4        /* goal-hit state on progress bars */
--orange: #d4954a      /* dinner pills, above-range glucose */
--purple: #9d6fff      /* snack meal pills */
--red: #d44a4a         /* delete actions, below-range glucose */
```

### Typography
- **Headers / numbers:** `Space Mono`
- **Body / labels / buttons:** `Inter`

### Progress Bars
- Track: `--green-4`
- Fill: `--green-1` (approaching goal) → `--blue` (goal met, CSS class `hit` added to `.progress`)

---

## Data Model

The app uses several localStorage keys, each with a clear responsibility:

| Key | Contents |
|---|---|
| `cfTrackerData_v1` | Main data: meals (by date), favorites, goals |
| `coimasDoseSettings_v1` | Insulin settings: ICR, glucose target, ISF (morning/afternoon/evening) |
| `cfTrackerSettings_v1` | App settings: Anthropic API key, GitHub credentials, last sync time |
| `cfLearningCorrections_v1` | AI photo estimate correction history (last 50 records) |
| `glucose:YYYY-MM-DD` | Raw glucose samples for that date (populated by the JSON import pipeline — not currently in use) |
| `report:YYYY-MM-DD` | Computed daily report JSON (pre-built for Claude prompt) |
| `rlib_YYYY-MM-DD` | Saved daily report HTML (from Claude artifact) |
| `rlib_w_YYYY-MM-DD` | Saved weekly report HTML |
| `rlib_meta_YYYY-MM-DD` | Extracted metadata for a daily report (TIR, protein, avg glucose) |
| `rlib_w_meta_YYYY-MM-DD` | Extracted metadata for a weekly report |
| `rlib_index` | Sorted array of daily report dates |
| `rlib_w_index` | Sorted array of weekly report dates |

### Main data blob (`cfTrackerData_v1`)
```json
{
  "meals": {
    "2026-04-26": [
      {
        "id": "m_1745123456789_abc12",
        "name": "Protein Smoothie",
        "time": "08:30",
        "starred": false,
        "macros": {
          "protein": 57, "fiber": 5, "carbs": 80, "calories": 680,
          "fat": 16, "satFat": 6, "sodium": 326, "sugar": 31,
          "calcium": 630, "vitD": 0, "insulin": 12
        },
        "doseLog": {
          "preBolus": "10min",
          "activityAfter": false,
          "iob": 2.5,
          "bsAtDose": 145
        },
        "aiEstimate": {  // present only when meal came from photo scan
          "protein": 55, "carbs": 78, "calories": 660,
          "confidence": "high", "note": "..."
        }
      }
    ]
  },
  "favorites": [
    {
      "id": "fav_protein_smoothie",
      "name": "Protein Smoothie",
      "macros": { /* same shape, insulin always 0 */ }
    }
  ],
  "goals": {
    "protein": 200,
    "fiber": 30,
    "calories": 2800
  }
}
```

### Key rules
- Meal dates: `YYYY-MM-DD` strings in **local time** (never UTC)
- Meal IDs: `m_` + timestamp + `_` + 5-char random string
- `netCarbs` always computed on the fly as `max(0, carbs - fiber)` — never stored
- Goals default to `{protein:200, fiber:30, calories:2800}` if missing
- Favorites strip insulin (always 0) — insulin is logged per meal at log time
- `doseLog` object optional — records pre-bolus timing, activity, IOB, and BS at dose time for calibration data
- `aiEstimate` object optional — original AI estimate kept so user corrections can be learned

---

## Tracked Fields

### Primary (progress bars)
| Field | Unit | Default Goal |
|---|---|---|
| Protein | g | 200 |
| Fiber | g | 30 |
| Calories | kcal | 2,800 |

### Secondary (stat cards)
Carbs · Net Carbs · Fat · Sat Fat · Sodium · Sugar · Vitamin D · Calcium · Insulin

---

## UI Structure

### Tabs
Five tabs: **Log** (default) · **Favorites** · **Calculator** · **Reports** · **Settings** (gear)

#### Log Tab
1. Date navigation (← date →)
2. Primary goal cards (Protein / Fiber / Calories) with progress bars
3. Secondary stat grid (8 cards)
4. Insulin card (full-width, green-tinted)
5. **Smart Paste** card with three input paths:
   - 📷 Photo button — AI macro estimation (Claude vision)
   - Paste text from Claude (manual prompt-then-paste flow)
   - "+ Add More" button appears after first partial-meal log
6. **Dose Panel** (appears after parse): meal chip, IOB block, BS input, time-of-day chip, exercise grid, pre-bolus timing, activity toggle, dose result with ± steppers, calculation breakdown drawer, "Log Meal + Units" / "Log without insulin" / "Cancel" actions
7. Manual meal entry form (collapsed behind "+ Log manually" toggle)
8. Today's meal list (edit/star/delete per row)
9. **Action card**:
   - Daily Report button (auto-copies JSON prompt)
   - Weekly Report button (opens 14-day picker, then auto-copies)
   - Glucose import status row (📋 Import button + setup help — infrastructure for future JSON pipeline, not currently active)

#### Favorites Tab
- Recent Meals card (top 5, deduped by name)
- Favorites list — each row: name, macro summary, USE / ✎ edit / × delete
- USE switches to Log tab and opens dose panel with favorite's macros pre-loaded

#### Calculator Tab
- Standalone dose calculator (correction-only mode when carbs = 0)
- IOB block — aggregated across all active Humalog doses in past 4 hours (Lantus excluded by name match)
- Night advisory appears when BS > 180 after 9pm
- "Log Insulin Only" card — units + reason picker (Correction / Lantus / Other) + optional BS at dose

#### Reports Tab
- Full-screen overlay with sandboxed iframe viewer
- Daily / Weekly mode tabs
- Header: ← back, prev/next/date-chip, index browse (☰), delete (🗑), import (+)
- Index panel — month-grouped, searchable, with TIR + macro summary pills per report
- Import sheet — paste Claude HTML, auto-detects date and daily-vs-weekly, warns on overwrite

#### Settings Tab
- API Keys — Anthropic key (for photo AI) and GitHub credentials (for auto-sync)
- **Insulin Settings** — ICR, glucose target, ISF (morning/afternoon/evening)
- **Goals** — protein, fiber, calories
- **Data Tools** — Export CSV, Export Full Backup, Smart Merge Import, Export/Import Reports, Move Tracker prompt, Clear Today, Reset All

---

## Dose Calculator

### Settings (user-configurable, stored in `coimasDoseSettings_v1`)
All values read from `getDoseSettings()` — no hardcoded constants:

```javascript
{
  icr: 10,            // 1u covers X grams of net carbs
  target: 120,        // glucose target in mg/dL
  isfMorning: 50,     // mg/dL one unit lowers BG, morning
  isfAfternoon: 50,
  isfEvening: 50
}
```

### Formula
```
net_carbs    = max(0, carbs - fiber)
carb_dose    = net_carbs / ICR    (0 for protein-only meals, carbs < 10g)
correction   = (BS - target) / ISF
IOB          = sum across all active Humalog doses in last 4 hours,
               each decaying linearly: dose × (1 - hours / 4)
exercise_adj = see below
recommended  = max(0, round_to_0.5(carb_dose + correction - IOB + exercise_adj))
```

Result is always shown alongside the calculation breakdown drawer and is editable via ± steppers (user can override the calculator).

### Exercise adjustments (applied to carb dose only, not correction)
- HIIT <1hr: no adjustment (cortisol active, BS is the signal)
- HIIT 1–3hr: −25% of carb dose
- HIIT 3–6hr: −20% of carb dose
- Sedentary: +1u (only on non-protein meals)
- None: standard

### Pre-bolus timing (informational, BS-aware)
- BS ≥ 120: pre-bolus window ~10 min
- BS 100–119: first bite (BS running low)
- BS < 100: with food (BS below range for pre-bolus)
- High fat (≥ 30g): with food or first bite regardless

### IOB rules
- Lantus excluded — any meal entry containing "lantus" in name is skipped
- Doses stacked: aggregate IOB summed across all active doses
- When prefilling dose panel, the calculator shows total IOB as a single equivalent dose at "0 hours ago"
- Night advisory: when BS > 180 after 9pm, warns about Lantus timing interaction

---

## Smart Paste Parser

**Format expected:**
```
Meal Name, Protein: 62, Fiber: 5, Carbs: 30, Calories: 860, Fat: 52, Sat Fat: 15, Sodium: 1660, Sugar: 5, Calcium: 320, Vit D: 0, Insulin: 0
```

**Parser rules:**
- Name = everything before the first recognized macro keyword
- Commas inside numbers stripped before splitting (`1,660` → `1660`)
- Units stripped (`52g`, `900mg`)
- Code fences stripped
- Missing fields default to 0
- `net carbs` in paste ignored — computed from carbs−fiber
- `cal` / `kcal` treated as `calories`
- Insulin supports half units (`1.5`)

---

## Photo / Barcode Macro Estimation

**Photo flow** (`scanPhotoMeal()`):
1. User taps 📷 Photo → opens iOS camera or photo library
2. Image resized to max 768px and converted to base64 JPEG
3. If browser supports `BarcodeDetector`, image scanned for UPC first — if found, OpenFoodFacts called and dose panel opens with product macros
4. Otherwise: optional description field shown ("e.g. 6oz chicken, cup of rice")
5. Parallel Haiku call checks if image matches an existing favorite — if so, suggests it
6. On submit: Claude Sonnet vision call returns macros JSON, dose panel opens
7. Past corrections (`cfLearningCorrections_v1`) are included in the system prompt so estimates improve over time

**Correction learning:**
- When a meal logged after photo scan has macros differing from the AI estimate by >2g protein / >3g carbs / >20 cal, a correction record is saved
- Last 50 corrections injected into subsequent photo-scan system prompts

---

## Glucose Pipeline

### Current flow (Claude pulls HealthKit directly)
1. User taps **⬡ Daily Report** in the tracker
2. `buildDailyReportPrompt()` builds a structured prompt containing:
   - Exact UTC timestamps for all 4 glucose pull windows (computed by `buildGlucoseWindows()`)
   - A workout pull window
   - Strict instructions: never combine windows, no HR pull, no steps pull, gap-fill rules
   - UTC→local time conversion formula
   - Full meal log with all macros and daily totals
3. Prompt is auto-copied to clipboard — user pastes into Claude
4. Claude pulls blood glucose samples from Apple HealthKit using the 4-window structure
5. Claude builds the HTML report artifact
6. User pastes the artifact into the tracker's Reports tab via **+**

### 4-window glucose pull structure
Four isolated HealthKit calls — no window crosses UTC midnight (prevents timeouts).

| Window | CDT Equivalent | Sample Limit |
|---|---|---|
| A | local midnight → 3pm | 220 |
| B1a | local 3pm → 7pm | 70 |
| B1b | local 7pm → 11pm | 70 |
| B2 | local 11pm → 5am next day | 90 |

If any window returns `samples_truncated: true`, Claude runs a gap-fill call before building the report. CDT vs CST auto-detected by `buildGlucoseWindows()`.

### JSON pipeline (in code, not currently in use)
The tracker contains a second path where glucose is imported first and the report JSON is pre-computed before Claude receives it. Functions involved: `buildReportJSON()`, `buildDailyReportPromptV2()`, `parseGlucoseClipboard()`, the 📋 Import button in the action card. This infrastructure exists but the Shortcut that feeds it is not working — do not assume it is active.

### Future path (not started)
When this app is converted to native iOS, glucose will be pulled one of two ways:
- **Apple HealthKit directly** — the app queries HealthKit natively, eliminating the Claude-pull step entirely. Most likely path if Dexcom integration takes time.
- **Dexcom API** — OAuth2 flow against `developer.dexcom.com`. Real-time streaming. Requires sandbox then production approval (2–6 weeks).

The decision depends on conversion timeline. Either way, Claude's role shifts to report-building only — it will receive pre-computed data and generate HTML, not pull raw samples.

### ICR calibration eligibility
A meal's effective ICR is computed only when ALL of these are false:

| Flag | Condition |
|---|---|
| `high_fat` | meal fat > 40g |
| `otf_day` | any workout recorded that day |
| `pre_bg_elevated` | pre-dose BG > 140 |
| `correction_given` | correction dose within 4 hours of meal |
| `stacking_risk` | previous Humalog dose within 3 hours |
| `response_unresolved` | glucose not back within 20% of pre-dose BG within 4 hours |

---

## Timezone System

All dates in local time, never bare UTC strings.

```javascript
toLocalISODate(date)        // Date → "YYYY-MM-DD" local
fromLocalISODate(str)       // "YYYY-MM-DD" → Date at local midnight
todayISO()                  // today as "YYYY-MM-DD" local
chicagoOffsetForDate(date)  // "-05:00" (CDT) or "-06:00" (CST)
buildGlucoseWindows(iso)    // returns all 4 UTC pull windows + workout window
                            // with exact timestamps computed from the report date
isoToChicagoDate(isoStr)    // ISO timestamp → "YYYY-MM-DD" in Chicago time
```

`buildGlucoseWindows()` auto-detects CDT vs CST and returns:
- Window A, B1a, B1b, B2 with exact UTC start/end timestamps
- Workout window with exact UTC start/end timestamps
- Timezone label string for the prompt

---

## Reports Library

Saved Claude-generated report artifacts (full HTML strings) live in localStorage under `rlib_*` keys. The Reports tab is a full-screen overlay rendered via a sandboxed `<iframe>` with `srcdoc`.

**Auto-resize mechanism:** an injected script in each report iframe posts its scroll height back to the parent via `postMessage`. The parent listens and sets `iframe.height`.

**Metadata extraction:** when a report is imported, regex patterns pull key stats (TIR, protein, avg glucose, low/high %) out of the HTML and store them in `rlib_meta_*` so the index/browse panel can show summary pills without re-parsing.

**Date detection:** the import sheet auto-detects the report date by scanning the pasted HTML for known patterns (`<title>Daily Report — ...`, "Week of YYYY-MM-DD", etc.).

---

## Glucose Chart System (in Reports)

Charts rendered in the Claude-generated report HTML using Canvas 2D API — no chart library.

### Day Panel (6am–6pm)
- x-axis: minutes 360–1080

### Night Panel (6pm–6am) — CRITICAL
- **Extended-minute space:** 6pm=1080, midnight=1440, 6am=1800
- Post-midnight samples: `ext_min = 1440 + local_min`
- **NEVER map night samples to 0–1440** — compresses axis, causes rendering bugs

### Glucose Line Colors
- Below 70: `#d44a4a` (red)
- 70–180: `#4dbb88` (green)
- Above 180: `#d4954a` (orange)
- Drawn in segments with crossing detection at 70 and 180

---

## GitHub Auto-Sync

Meal data and saved reports sync automatically to a private GitHub repo (`nutrition-tracker`) after each save, debounced at 30 seconds. Credentials live in `cfTrackerSettings_v1`.

Entry point: `githubSync()`. Manual trigger available in Settings ("⟳ Sync Now").

Files pushed:
- `backup/meals.json` — full main data blob with timestamp
- `reports/daily-YYYY-MM-DD.html` — last 30 daily reports
- `reports/weekly-YYYY-MM-DD.html` — last 12 weekly reports

---

## Export CSV

Columns: `date, time, name, protein_g, fiber_g, carbs_g, net_carbs_g, calories, fat_g, sat_fat_g, sodium_mg, sugar_g, calcium_mg, vit_d_iu, insulin_u, bs_at_dose`

---

## How to Adapt for a New User

### Change goals
Edit in Settings tab → Goals. Or change defaults in code:
```javascript
const DEFAULT_GOALS = { protein: 200, fiber: 30, calories: 2800 };
```

### Change insulin settings
All adjustable in Settings tab → Insulin Settings: ICR, glucose target, and time-of-day ISF.  
No code changes needed — these live in `coimasDoseSettings_v1`.

### Change tracked fields
1. Add/remove from `blankMacros()`
2. Update Smart Paste parser (`MACRO_KEYS` + parsing branch)
3. Update HTML cards in secondary grid
4. Update `buildDailyReportPrompt()` and `buildReportJSON()` meal logs
5. Update CSV export columns
6. Update edit modal fields

### Change timezone
Replace `chicagoOffsetForDate()` and `buildGlucoseWindows()` with functions targeting the new timezone using `Intl.DateTimeFormat`.

---

## Known Issues / Bugs Fixed

| Bug | Fix |
|---|---|
| Comma-in-number parsing (`1,660mg` splitting wrong) | Strip digit-comma-digit before splitting |
| Night chart x-axis compression (6:48pm rendering near midnight) | Extended-minute system (6pm=1080, midnight=1440, 6am=1800) |
| UTC drift with bare date strings in HealthKit pulls | 4-window UTC system with exact computed timestamps in prompt |
| iOS zoom on textarea focus | `font-size: 16px` on textareas + `maximum-scale=1.0` on viewport |
| Lantus counting toward Humalog IOB | Name filter in `getLastInsulinDose()` and `getActiveIob()` |
| Black box at bottom of report iframe | `r-content` overflow-y auto, frame-wrap no longer absolutely positioned |
| Stacked Humalog doses being undercounted in IOB | `getActiveIob()` sums all doses in past 4 hours |
| Reports `+` button paste sheet hiding behind overlay | Sheet `z-index` raised above reports UI layer |

---

## Deployment

Single HTML file, host anywhere serving HTTPS:
- **GitHub Pages** (current) — free, requires GitHub account
- **iCloud Drive** — no hosting, open in Safari → Add to Home Screen

Data lives in `localStorage` — device and browser specific. Cross-device sync handled via the GitHub auto-sync feature. Migration to a new device handled by the "Move Tracker" prompt builder in Settings → Data Tools.
