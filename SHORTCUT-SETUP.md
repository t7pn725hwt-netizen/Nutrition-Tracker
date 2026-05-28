# Glucose Shortcut Setup Guide

_Last updated: 2026-05-28_

This Shortcut pulls blood glucose from Apple Health and copies it to your clipboard so the tracker can import it with one tap.

---

## What it does

Queries HealthKit for blood glucose samples covering the full report day, formats them as JSON, and copies the result to your clipboard. The tracker reads the clipboard when you tap **📋 Import**.

---

## Build the Shortcut — step by step

Open the **Shortcuts** app and create a new Shortcut. Add these actions in order:

### Step 1 — Find health samples
- Action: **Find Health Samples**
- Type: **Blood Glucose**
- Start Date: **Start of Today**
- End Date: **End of Today**
- Sort by: Start Date (ascending)
- Limit: none

> If you're pulling data for yesterday, change Start/End to "Start of Yesterday" / "End of Yesterday". You can make this a menu at the top if you want both options.

### Step 2 — Set up an empty list
- Action: **Set Variable**
- Variable name: `Results`
- Value: *(empty list — tap the variable field and leave it blank)*

### Step 3 — Repeat with each sample
- Action: **Repeat with Each Item in** Health Samples

### Step 4 — Get the timestamp (inside repeat)
- Action: **Get Details of Health Sample**
- Detail: **Start Date**
- Save result to variable: `SampleDate`

- Action: **Format Date**
- Date: `SampleDate`
- Format: **Custom**
- Custom format string: `yyyy-MM-dd'T'HH:mm:ssZZZZZ`
- Save result to variable: `ts`

> The format string `yyyy-MM-dd'T'HH:mm:ssZZZZZ` produces ISO 8601 with timezone offset, e.g. `2026-05-23T07:03:00-05:00`. This is the format the tracker expects.

### Step 5 — Get the value (inside repeat)
- Action: **Get Details of Health Sample**
- Detail: **Value**
- Save result to variable: `val`

### Step 6 — Build the JSON line (inside repeat)
- Action: **Text**
- Content: `{"t":"[ts]","v":[val]}`

  *(tap to insert variables — use the variable picker for `ts` and `val`)*

- Action: **Add to Variable**
- Variable: `Results`
- Item: the Text result above

### Step 7 — End Repeat

### Step 8 — Copy to clipboard
- Action: **Get Text** from `Results`
- Action: **Copy to Clipboard**

### Step 9 — Optional: Open tracker
- Action: **Open URL**
- URL: `https://t7pn725hwt-netizen.github.io/Nutrition-Tracker/`

---

## Testing

After running the Shortcut, open a notes app and paste. You should see something like:

```
{"t":"2026-05-23T00:03:00-05:00","v":118}
{"t":"2026-05-23T00:08:00-05:00","v":121}
{"t":"2026-05-23T00:13:00-05:00","v":124}
...
```

One JSON object per line, one per glucose reading (~288 for a full day at 5-min intervals).

If the clipboard is empty, check that Health permissions are granted: **Settings → Privacy & Security → Health → Shortcuts → Blood Glucose → Read**.

---

## Adding to your Home Screen

1. Tap the Shortcut name at the top → **Details**
2. Toggle **Add to Home Screen**
3. Give it a name like "Glucose Pull"
4. Place it next to your tracker icon

---

## Daily workflow

1. Tap **Glucose Pull** shortcut → runs, copies data, opens tracker
2. Tap **📋 Import** in the tracker → reads clipboard → status turns green
3. Tap **⬡ Daily Report** → copies JSON prompt
4. Paste in Claude → report builds from pre-computed data, no HealthKit pull needed

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| "No valid glucose data found" | Check clipboard in Notes — if empty, check Health permissions |
| Status shows 0 pts | Dexcom may have a sync delay — wait 5 min and re-run |
| Wrong date's data | Change Shortcut Start/End to match report date |
| Fewer than 200 pts | Expected if sensor is in warmup (~2hr every 15 days). Tracker will flag `sensor_gap` but continues normally |

---

## Future: Swift wrapper

When the tracker is wrapped in a native iOS app, this Shortcut becomes unnecessary. The wrapper will call HealthKit directly when you tap **⬡ Daily Report** — no clipboard step, no separate app. The same JSON format will be used, so all downstream processing stays identical.
