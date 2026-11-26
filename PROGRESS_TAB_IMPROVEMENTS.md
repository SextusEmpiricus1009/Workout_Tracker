# Progress Tab — Implementation Plan

**File:** `gym_tracker.html`  
**Scope:** All changes are inside `renderProgress()` and its helpers, plus new helper functions and CSS.  
**Data already available:** `logs` (exercise history), `sessions` (daily session records), `goals`, `program` (weekly plan)

---

## Implementation Order

Features are ordered by value-to-effort ratio. Each builds on the previous ones where possible.

1. Personal Records (PRs)
2. All-Time Records Panel
3. Estimated 1RM Trend
4. Volume Chart Per Exercise
5. Muscle Group Balance View
6. Workout Type Split
7. Running — Weekly Distance Chart
8. Heart Rate Zone Breakdown
9. Yearly Heatmap
10. Goal Progress on Progress Tab

---

## Feature 1 — Personal Records (PRs)

**What:** Flag when a logged weight is the highest ever for that exercise. Show a "Recent PRs" card at the top of the Progress tab, and add a PR badge to log entries in the exercise history.

**Data source:** `logs[exerciseName]` — each entry has `{ date, weight, reps, rpe, notes }`. Weight is stored as a string (e.g. `"80"`), parsed with `parseFloat`.

**Helper to add:**
```js
function getPRs() {
  // Returns { exerciseName: { weight, date } } — the max-weight entry per exercise
  const prs = {};
  Object.entries(logs).forEach(([name, entries]) => {
    let best = null;
    entries.forEach(e => {
      const w = parseFloat(e.weight);
      if (!isNaN(w) && (!best || w > best.weight)) best = { weight: w, date: e.date, reps: e.reps };
    });
    if (best) prs[name] = best;
  });
  return prs;
}
```

**Render changes:**
- Add a "Recent PRs" section after the stats row — shows the 5 most recent PRs (sorted by date desc).
- In `renderProgress()` exercise log entries: compare `entry.weight` to `prs[name].weight`; if equal, append a `<span class="pr-badge">PR</span>`.

**CSS to add:**
```css
.pr-badge { background: var(--accent); color: #0d0d0f; font-size: 9px; font-weight: 700; letter-spacing: 1px; padding: 1px 5px; border-radius: 3px; margin-left: 4px; vertical-align: middle; }
.pr-card { background: var(--surface); border: 1px solid var(--border); border-left: 3px solid var(--accent); border-radius: 8px; padding: 12px 14px; margin-bottom: 12px; }
.pr-list-item { display: flex; justify-content: space-between; padding: 5px 0; border-bottom: 1px solid var(--border); font-size: 13px; }
.pr-list-item:last-child { border-bottom: none; }
```

---

## Feature 2 — All-Time Records Panel

**What:** A dedicated "Records" card showing: longest streak ever, most sessions in a single week, total volume lifted (all time), fastest run pace, longest run distance.

**Data source:** `sessions` object (keys are date strings), `logs`, run sessions from `getRunSessions()`.

**Helper to add:**
```js
function getAllTimeRecords() {
  const dates = Object.keys(sessions).filter(d =>
    getSessionsForDate(d).some(s => (s.exLogs && s.exLogs.length > 0) || s.entryType === 'raceDay' || s.entryType === 'ballGame')
  ).sort();

  // Longest streak (weeks)
  let longestStreak = 0, curStreak = 0;
  // iterate weeks backwards same as getStats() streak logic but track max

  // Most sessions in a single week
  let maxWeekSessions = 0;
  // group dates by ISO week, find max count

  // Total volume (kg × reps across all logs)
  let totalVolume = 0;
  Object.values(logs).forEach(entries => {
    entries.forEach(e => {
      const w = parseFloat(e.weight) || 0;
      const r = parseInt(e.reps) || 0; // reps may be "3x10" — try to parse sets×reps
      totalVolume += w * r;
    });
  });

  // Fastest pace & longest distance from run sessions
  const runs = getRunSessions();
  const paces = runs.filter(r => r.pace).map(r => parsePace(r.pace)).filter(Boolean);
  const bestPace = paces.length ? Math.min(...paces) : null;
  const distances = runs.filter(r => r.distance).map(r => r.distance);
  const longestRun = distances.length ? Math.max(...distances) : null;

  return { longestStreak, maxWeekSessions, totalVolume, bestPace, longestRun };
}
```

**Render:** A `<div class="pr-card">` grid with 5 stat tiles, placed just below the main stats row.

---

## Feature 3 — Estimated 1RM Trend

**What:** For the selected strength exercise, add a second chart (or toggle) showing estimated 1RM over time using the Epley formula: `e1RM = weight × (1 + reps / 30)`. This normalises across varying rep ranges and gives a truer strength trendline.

**Data source:** `logs[exerciseName]` — entries with both `weight` and parseable `reps`.

**Reps parsing:** The `reps` field is a free-text string like `"3x10"`, `"10"`, or `"8 reps"`. Parse with:
```js
function parseReps(repsStr) {
  if (!repsStr) return null;
  // Handle "3x10" format — use the rep count (second number)
  const setsReps = repsStr.match(/\d+\s*[xX×]\s*(\d+)/);
  if (setsReps) return parseInt(setsReps[1]);
  const plain = repsStr.match(/\d+/);
  return plain ? parseInt(plain[0]) : null;
}

function calcE1RM(weight, repsStr) {
  const w = parseFloat(weight);
  const r = parseReps(repsStr);
  if (!w || !r || r <= 1) return null;
  return Math.round(w * (1 + r / 30) * 10) / 10;
}
```

**Render changes:**
- Add a toggle above the exercise chart: `[ Weight ] [ Est. 1RM ]`
- When "Est. 1RM" is active, filter entries where both weight and reps exist, compute e1RM, and pass to a modified `drawChart()` that accepts a custom data series and y-axis label.
- Chart title becomes `"Est. 1RM over time — {name}"`.

**State variable to add:** `let progressChartMode = 'weight'; // 'weight' | 'e1rm'`

---

## Feature 4 — Volume Chart Per Exercise

**What:** A third chart mode (or separate chart) showing total session volume (sets × reps × weight) per workout date for the selected exercise. Useful for spotting deload weeks and progressive overload patterns that raw weight misses.

**Data source:** `logs[exerciseName]` — group by date, sum `weight × reps` across all drop sets on the same day.

**Volume calculation:**
```js
function getVolumeData(exName) {
  const entries = logs[exName] || [];
  const byDate = {};
  entries.forEach(e => {
    const w = parseFloat(e.weight) || 0;
    const r = parseReps(e.reps) || 0;
    // Detect sets from "3x10" format
    const setsMatch = (e.reps || '').match(/^(\d+)\s*[xX×]/);
    const sets = setsMatch ? parseInt(setsMatch[1]) : 1;
    byDate[e.date] = (byDate[e.date] || 0) + w * r * sets;
  });
  return Object.entries(byDate).sort(([a],[b]) => a.localeCompare(b))
    .map(([date, vol]) => ({ date, val: Math.round(vol) }));
}
```

**Render:** Add `[ Volume ]` as a third toggle alongside Weight and Est. 1RM. Volume chart reuses the same `drawChart()` canvas with `kg` replaced by `vol` in the y-axis label.

---

## Feature 5 — Muscle Group Balance View

**What:** A horizontal bar chart showing the distribution of training across muscle groups for the selected time period (week / month / all). Uses the `muscle` tag already stored on exercises in the `program`.

**Data source:** `sessions` (for exercise names logged per date) cross-referenced against `program` (which has `muscle` or `category` tags per exercise). Fallback: derive group from exercise name keywords (e.g. "squat" → legs, "bench" → push).

**Helper to add:**
```js
function getMuscleGroupData(filterDates) {
  const counts = {};
  filterDates.forEach(dateStr => {
    getSessionsForDate(dateStr).forEach(s => {
      (s.exLogs || []).forEach(ex => {
        const group = getExerciseMuscleGroup(ex.name); // lookup from program or keyword match
        counts[group] = (counts[group] || 0) + 1;
      });
    });
  });
  return counts;
}

function getExerciseMuscleGroup(name) {
  // Check program first
  for (const day of program) {
    const match = (day.exercises || []).find(e => e.name === name);
    if (match && match.muscle) return match.muscle;
  }
  // Keyword fallback
  const n = name.toLowerCase();
  if (/squat|leg press|lunge|deadlift|rdl|hamstring|quad|calf/.test(n)) return 'Legs';
  if (/bench|push|chest|fly|dip|shoulder|press|tricep/.test(n)) return 'Push';
  if (/row|pull|lat|back|bicep|curl/.test(n)) return 'Pull';
  if (/plank|crunch|ab|core|sit.up/.test(n)) return 'Core';
  if (/run|cardio|bike|swim/.test(n)) return 'Cardio';
  return 'Other';
}
```

**Render:** Horizontal bar chart drawn on a `<canvas>`, with bars coloured per muscle group. Placed below session history, above the running section.

---

## Feature 6 — Workout Type Split

**What:** A small bar chart or pill summary showing what proportion of sessions this month (or selected period) were Strength / Aerobic / Hybrid / Recovery / Other.

**Data source:** `sessions` — each session has `dayType` (`"strength"`, `"aerobic"`, `"hybrid"`, `"recovery"`, etc.).

**Implementation:** Simple count aggregation. Rendered as a horizontal stacked bar or labelled pill row — no canvas needed, can be pure CSS.

```html
<div class="type-split-bar">
  <div class="type-segment strength" style="width: 45%"></div>
  <div class="type-segment aerobic" style="width: 30%"></div>
  <div class="type-segment hybrid" style="width: 15%"></div>
  <div class="type-segment recovery" style="width: 10%"></div>
</div>
```

CSS widths computed dynamically. Legend labels sit below the bar. Respects the active `historyFilter` period.

---

## Feature 7 — Running: Weekly Distance Chart

**What:** A bar chart showing total km run per week over the last 12 weeks. Makes training load progression (and cutback weeks) immediately visible.

**Data source:** `getRunSessions()` — each run has `{ date, distance }`. Group by ISO week, sum distances.

**Helper to add:**
```js
function getWeeklyRunDistance(nWeeks = 12) {
  const runs = getRunSessions();
  const weeks = [];
  const now = new Date();
  for (let w = nWeeks - 1; w >= 0; w--) {
    const mon = new Date(now);
    mon.setDate(now.getDate() - (now.getDay()+6)%7 - w*7);
    const sun = new Date(mon); sun.setDate(mon.getDate() + 6);
    const wStart = fmtLocal(mon), wEnd = fmtLocal(sun);
    const dist = runs.filter(r => r.date >= wStart && r.date <= wEnd)
                     .reduce((sum, r) => sum + (r.distance || 0), 0);
    weeks.push({ label: wStart.slice(5), dist: Math.round(dist * 10) / 10 });
  }
  return weeks;
}
```

**Render:** A canvas bar chart inside the Running section, replacing the current static weekly km stat card (or sitting below it). Bars coloured `var(--green)`, with km labels above each bar.

---

## Feature 8 — Heart Rate Zone Breakdown

**What:** For runs with HR data, show what proportion of runs fell into each HR zone. Zones are user-configurable (stored in `localStorage` under `'hrZones'`), with sensible defaults based on a 180bpm max HR.

**Default zones:**
```js
const DEFAULT_HR_ZONES = [
  { name: 'Z1 Recovery', max: 114, color: '#6dbcdc' },
  { name: 'Z2 Aerobic',  max: 133, color: '#6ddc9a' },
  { name: 'Z3 Tempo',    max: 152, color: '#e8ff47' },
  { name: 'Z4 Threshold',max: 162, color: '#ff9f6b' },
  { name: 'Z5 Max',      max: 999, color: '#e05f5f' }
];
```

**Implementation:**
- Load/save zones from `localStorage`.
- Classify each run's avg HR into a zone.
- Display as a donut-style or horizontal stacked bar showing zone distribution.
- A small "Edit Zones" toggle reveals threshold input fields.

**Note:** This uses avg HR per run (already stored), not per-second HR data (not available). This is noted in the UI with a "based on avg HR" disclaimer.

---

## Feature 9 — Yearly Heatmap

**What:** A GitHub-style 52-column × 7-row grid where each cell is one day. Cells are coloured by workout intensity: no session (dark), logged session (accent colour), with opacity scaled by exercise count or session type.

**Data source:** `sessions` — check which dates have logged sessions. No new data required.

**Implementation:**
- Generate 365 days from today backwards.
- For each day, check `sessions[dateStr]`; compute a "heat" level (0 = rest, 1 = recovery/light, 2 = moderate, 3 = heavy/strength).
- Render as an SVG or CSS grid (SVG preferred for fine cell sizing).
- Month labels above each column group.
- Tooltip on hover showing date + session name.

**Heat level mapping:**
```js
function getHeatLevel(dateStr) {
  const daySessions = getSessionsForDate(dateStr).filter(s => s.exLogs?.length > 0 || s.entryType);
  if (!daySessions.length) return 0;
  const type = daySessions[0].dayType || '';
  if (type === 'recovery') return 1;
  if (type === 'aerobic') return 2;
  return 3; // strength, hybrid, race-day, etc.
}
```

**CSS cells:**
```css
.heatmap-cell { width: 11px; height: 11px; border-radius: 2px; }
.heat-0 { background: var(--surface2); }
.heat-1 { background: rgba(232,255,71,0.2); }
.heat-2 { background: rgba(232,255,71,0.5); }
.heat-3 { background: var(--accent); }
```

---

## Feature 10 — Goal Progress on Progress Tab

**What:** A compact "Active Goals" section on the Progress tab showing goals that are not yet completed, with a progress indicator where possible.

**Data source:** `goals` array — each goal has `{ text, deadline, done }`.

**Progress estimation:** Most goals are free-text and can't be auto-measured. For goals whose text matches an exercise name in `logs`, show "current best vs goal" if the goal text contains a number (e.g. "Squat 100kg"). Otherwise show time remaining to deadline.

**Render:**
```js
function renderGoalProgress() {
  const active = goals.filter(g => !g.done);
  if (!active.length) return '';
  let html = `<div class="section-label">Active Goals</div>`;
  active.forEach(g => {
    const daysLeft = g.deadline ? Math.ceil((new Date(g.deadline) - new Date()) / 86400000) : null;
    const daysLabel = daysLeft !== null
      ? daysLeft > 0 ? `${daysLeft}d left` : `${Math.abs(daysLeft)}d overdue`
      : '';
    const overdue = daysLeft !== null && daysLeft < 0;
    html += `<div class="goal-progress-card">
      <div class="goal-progress-text">${escapeHtml(g.text)}</div>
      ${daysLabel ? `<div class="goal-progress-deadline ${overdue ? 'overdue' : ''}">${daysLabel}</div>` : ''}
    </div>`;
  });
  return html;
}
```

**Placement:** Between the stats row and the calendar — keeps goals visible at the top of the screen.

---

## CSS Additions Summary

All new CSS is added inside the existing `<style>` block. Key new classes:

| Class | Purpose |
|---|---|
| `.pr-badge` | Yellow PR badge on log entries |
| `.pr-card` | Left-accented card for PR / records sections |
| `.pr-list-item` | Row inside PR card |
| `.type-split-bar` | Stacked bar container for workout type split |
| `.type-segment` | Coloured segment inside the stacked bar |
| `.heatmap-wrap` | Container for the yearly heatmap grid |
| `.heatmap-cell` | Individual day cell (11×11px) |
| `.heat-0/1/2/3` | Heat level colours |
| `.goal-progress-card` | Compact goal row on Progress tab |
| `.goal-progress-deadline` | Deadline label (red when overdue) |
| `.chart-toggle` | Button row to switch chart modes (Weight/1RM/Volume) |
| `.chart-toggle-btn` | Individual toggle button |
| `.chart-toggle-btn.active` | Active state |

---

## State Variables to Add

```js
let progressChartMode = 'weight'; // 'weight' | 'e1rm' | 'volume'
let hrZones = null; // loaded from localStorage or DEFAULT_HR_ZONES
```

---

## Render Order in `renderProgress()`

After all features are implemented, `renderProgress()` will build sections in this order:

1. Stats row (existing)
2. All-Time Records Panel *(new)*
3. Active Goals callout *(new)*
4. Workout Type Split bar *(new)*
5. Calendar (existing)
6. Calendar day detail (existing, conditional)
7. Session History (existing)
8. Muscle Group Balance chart *(new)*
9. Running section — stats + weekly distance chart + pace/HR + efficiency + run history *(extended)*
10. HR Zone Breakdown *(new, inside running section)*
11. Exercise Progress — pill selector + chart with mode toggle + log entries with PR badges *(extended)*
12. Recent PRs card *(new)*
13. Yearly Heatmap *(new)*

---

## Notes

- All canvas charts reuse the existing `drawChart()` pattern (DPR-aware, manual canvas 2D API). No new chart library needed.
- HR zones are stored in `localStorage` only (not synced to Firebase) — they're a device preference, not user data.
- The reps parser (`parseReps()`) is shared by Features 3 and 4 — define it once above both.
- All new sections respect the existing dark/light theme via CSS variables.
- No breaking changes to the existing data schema — all new features read existing data.
