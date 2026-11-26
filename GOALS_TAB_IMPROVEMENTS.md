# Goals Tab — Implementation Plan

**File:** `gym_tracker.html`  
**Scope:** `screen-goals` HTML, `renderGoals()`, `addGoal()`, `toggleGoal()`, `deleteGoal()`, new helpers, new CSS, and the `goals[]` data schema.

---

## Current State

**Data schema (existing):**
```js
goals = [{ id, text, deadline, done }]
```

**UI (existing):**
- Single form: text input + date picker + Add button
- Cards: done checkbox, goal text, deadline label, delete button
- No categories, no measurable targets, no filtering, no history

---

## Implementation Order

| # | Feature | Effort | Value |
|---|---|---|---|
| 1 | Data schema migration | Low | Blocker for everything else |
| 2 | Filter tabs: Active / Done / All | Low | High |
| 3 | Deadline urgency colouring | Low | High |
| 4 | Priority level | Low | Medium |
| 5 | Goal categories | Low | Medium |
| 6 | Measurable goals + auto-linking | High | Very High |
| 7 | Archive section for completed goals | Low | Medium |
| 8 | Habit goals (weekly frequency target) | Medium | High |
| 9 | Notes / check-ins per goal | Medium | Medium |

---

## Feature 1 — Data Schema Migration

**New schema:**
```js
{
  id: Number,           // Date.now() — unchanged
  text: String,         // goal description — unchanged
  deadline: String,     // YYYY-MM-DD — unchanged
  done: Boolean,        // unchanged
  doneAt: String,       // NEW — YYYY-MM-DD when marked done (for archive)
  category: String,     // NEW — 'strength' | 'running' | 'habit' | 'race' | 'body' | 'other'
  priority: String,     // NEW — 'high' | 'medium' | 'low'
  goalType: String,     // NEW — 'binary' | 'measurable' | 'habit'
  target: Number,       // NEW — numeric target value (e.g. 100 for 100kg)
  unit: String,         // NEW — 'kg' | 'km' | 'min/km' | 'reps' | 'x/week'
  linkExercise: String, // NEW — exercise name to auto-link (for measurable strength goals)
  notes: Array,         // NEW — [{ date: 'YYYY-MM-DD', text: String }]
  weeklyTarget: Number, // NEW — for habit goals: sessions per week
}
```

**Migration:** All existing goals are backward-compatible. New fields default to `undefined` / `null` and are treated as optional throughout. `addGoal()` is extended to populate new fields from the expanded form.

**No data is lost.** Existing goals continue to render as `binary` goals with no category.

---

## Feature 2 — Filter Tabs: Active / Done / All

**State variable to add:**
```js
let goalsFilter = 'active'; // 'active' | 'done' | 'all'
```

**HTML change** — Replace `<div id="goals-list">` with:
```html
<div class="filter-tabs" id="goals-filter-tabs">
  <button class="filter-tab active" onclick="setGoalsFilter('active',this)">Active</button>
  <button class="filter-tab" onclick="setGoalsFilter('done',this)">Done</button>
  <button class="filter-tab" onclick="setGoalsFilter('all',this)">All</button>
</div>
<div id="goals-list"></div>
```

**Function to add:**
```js
function setGoalsFilter(f, btn) {
  goalsFilter = f;
  document.querySelectorAll('#goals-filter-tabs .filter-tab').forEach(b => b.classList.remove('active'));
  btn.classList.add('active');
  renderGoals();
}
```

**`renderGoals()` change:** Apply filter before rendering:
```js
const filtered = goals.filter(g =>
  goalsFilter === 'active' ? !g.done :
  goalsFilter === 'done'   ? g.done  : true
);
```

Uses existing `.filter-tab` and `.filter-tabs` CSS — no new CSS needed.

---

## Feature 3 — Deadline Urgency Colouring

**Logic:** For each goal with a deadline, compute `daysLeft`:
```js
function getGoalDaysLeft(g) {
  if (!g.deadline || g.done) return null;
  return Math.ceil((new Date(g.deadline + 'T12:00:00') - new Date()) / 86400000);
}
```

**Colour tiers:**
| Days left | Colour | Class |
|---|---|---|
| > 14 | `var(--muted)` | `.deadline-ok` |
| 7–14 | `#e8c84a` (amber) | `.deadline-soon` |
| 1–6 | `#ff9f6b` (orange) | `.deadline-urgent` |
| 0 | `var(--accent)` (yellow) | `.deadline-today` |
| < 0 | `#dc6d6d` (red) | `.deadline-overdue` |

**Display:** Replace the static "Target: YYYY-MM-DD" label with:
```html
<div class="goal-deadline deadline-urgent">6d left · 2026-07-06</div>
```

**CSS to add:**
```css
.deadline-ok      { color: var(--muted); }
.deadline-soon    { color: #e8c84a; }
.deadline-urgent  { color: #ff9f6b; }
.deadline-today   { color: var(--accent); font-weight: 600; }
.deadline-overdue { color: #dc6d6d; }
```

---

## Feature 4 — Priority Level

**UI in add form:** A 3-button toggle below the deadline:
```html
<div class="section-label" style="...">Priority</div>
<div class="chart-toggle" id="goal-priority-toggle">
  <button class="chart-toggle-btn" data-p="high"   onclick="setGoalPriority('high',this)">High</button>
  <button class="chart-toggle-btn active" data-p="medium" onclick="setGoalPriority('medium',this)">Medium</button>
  <button class="chart-toggle-btn" data-p="low"    onclick="setGoalPriority('low',this)">Low</button>
</div>
```

State variable: `let newGoalPriority = 'medium';`

**Sort order in `renderGoals()`:**
```js
const PRIORITY_ORDER = { high: 0, medium: 1, low: 2 };
filtered.sort((a, b) => {
  // Overdue first, then by priority, then by deadline
  const aOver = getGoalDaysLeft(a) < 0 ? -1 : 0;
  const bOver = getGoalDaysLeft(b) < 0 ? -1 : 0;
  if (aOver !== bOver) return aOver - bOver;
  const pA = PRIORITY_ORDER[a.priority || 'medium'];
  const pB = PRIORITY_ORDER[b.priority || 'medium'];
  if (pA !== pB) return pA - pB;
  return (a.deadline || 'z').localeCompare(b.deadline || 'z');
});
```

**Priority indicator on card:** A small left-border colour:
```css
.goal-card.priority-high   { border-left: 3px solid #dc6d6d; }
.goal-card.priority-medium { border-left: 3px solid var(--accent); }
.goal-card.priority-low    { border-left: 3px solid var(--border); }
```

---

## Feature 5 — Goal Categories

**UI in add form:** A horizontal scroll of pill buttons (reusing `.ex-pill` / `.exercise-select-scroll`):
```html
<div class="exercise-select-scroll">
  <button class="ex-pill active" onclick="setGoalCategory('strength',this)">💪 Strength</button>
  <button class="ex-pill" onclick="setGoalCategory('running',this)">🏃 Running</button>
  <button class="ex-pill" onclick="setGoalCategory('habit',this)">🔁 Habit</button>
  <button class="ex-pill" onclick="setGoalCategory('race',this)">🏁 Race</button>
  <button class="ex-pill" onclick="setGoalCategory('body',this)">⚖️ Body</button>
  <button class="ex-pill" onclick="setGoalCategory('other',this)">📌 Other</button>
</div>
```

State variable: `let newGoalCategory = 'strength';`

**On card:** A small badge using existing `.day-type-badge` pattern:
```js
const CATEGORY_LABELS = { strength:'💪', running:'🏃', habit:'🔁', race:'🏁', body:'⚖️', other:'📌' };
// rendered inline next to goal name
```

No new CSS needed — reuses existing pill and badge styles.

---

## Feature 6 — Measurable Goals + Auto-linking

This is the most complex feature. It adds a numeric target and automatically reads current progress from existing data.

### 6a — Extended Add Form

When `goalType === 'measurable'` is selected, reveal extra fields:
```html
<div id="goal-measurable-fields" style="display:none">
  <label>Target value</label>
  <input id="goal-target" type="number" placeholder="e.g. 100">
  <label>Unit</label>
  <div class="chart-toggle">
    <button class="chart-toggle-btn active" data-unit="kg">kg</button>
    <button class="chart-toggle-btn" data-unit="km">km</button>
    <button class="chart-toggle-btn" data-unit="reps">reps</button>
    <button class="chart-toggle-btn" data-unit="min/km">min/km</button>
  </div>
  <label>Link to exercise (optional)</label>
  <input id="goal-link-exercise" placeholder="e.g. Squat" list="exercise-datalist">
  <datalist id="exercise-datalist"></datalist>
</div>
```

The datalist is populated from `getAllExercises()` when the form opens.

### 6b — Auto-link Resolution

```js
function resolveGoalProgress(g) {
  // Returns { current, pct } or null
  if (g.goalType !== 'measurable' || !g.target) return null;

  if (g.unit === 'kg' && g.linkExercise) {
    // Pull max weight from logs
    const exLogs = logs[g.linkExercise] || [];
    const maxW = Math.max(...exLogs.map(e => parseFloat(e.weight) || 0).filter(Boolean), 0);
    return maxW > 0 ? { current: maxW, pct: Math.min(maxW / g.target * 100, 100) } : null;
  }

  if (g.unit === 'km') {
    // Longest run distance
    const runs = getRunSessions();
    const best = Math.max(...runs.map(r => r.distance || 0), 0);
    return best > 0 ? { current: best, pct: Math.min(best / g.target * 100, 100) } : null;
  }

  if (g.unit === 'min/km') {
    // Best (lowest) pace — progress = how close current pace is to target
    const runs = getRunSessions();
    const paces = runs.filter(r => r.pace).map(r => parsePace(r.pace)).filter(Boolean);
    if (!paces.length) return null;
    const best = Math.min(...paces); // lower = faster
    const target = parsePace(String(g.target)); // target stored as seconds
    return target ? { current: formatPace(best), pct: Math.min(target / best * 100, 100) } : null;
  }

  if (g.unit === 'reps' && g.linkExercise) {
    // Best rep count
    const exLogs = logs[g.linkExercise] || [];
    const maxR = Math.max(...exLogs.map(e => parseReps(e.reps) || 0), 0);
    return maxR > 0 ? { current: maxR, pct: Math.min(maxR / g.target * 100, 100) } : null;
  }

  return null;
}
```

### 6c — Progress Bar on Card

```html
<div class="goal-progress-bar-wrap">
  <div class="goal-progress-bar" style="width: 72%"></div>
</div>
<div class="goal-progress-label">Current: 72kg → Target: 100kg</div>
```

**CSS to add:**
```css
.goal-progress-bar-wrap { background: var(--surface2); border-radius: 4px; height: 6px; margin: 8px 0 4px; overflow: hidden; }
.goal-progress-bar      { height: 100%; background: var(--accent); border-radius: 4px; transition: width 0.4s ease; }
.goal-progress-label    { font-size: 10px; color: var(--muted); }
```

Auto-mark done when `pct >= 100` on next data load (optional, with a toast-style confirmation).

---

## Feature 7 — Archive Section for Completed Goals

Instead of mixing done goals into the main list (or hiding them entirely), show a collapsible "Completed" section at the bottom of the `done` and `all` filter views.

**When filter = 'active':** Only active goals show. A small "X completed" link at the bottom links to the Done view.

**When filter = 'done':** Shows completed goals sorted by `doneAt` descending (most recently completed first). Each card shows the completion date.

**`toggleGoal()` change:** Record completion date:
```js
function toggleGoal(i) {
  goals[i].done = !goals[i].done;
  goals[i].doneAt = goals[i].done ? todayStr() : null;
  save('goals', goals);
  renderGoals();
}
```

No new CSS needed — completed cards reuse existing strikethrough style with `doneAt` appended to the deadline line.

---

## Feature 8 — Habit Goals

A `goalType: 'habit'` goal tracks "do X sessions per week." Uses existing `sessions` data to auto-calculate completion.

**Add form:** When `goalType === 'habit'` is selected, show:
```html
<div id="goal-habit-fields" style="display:none">
  <label>Sessions per week</label>
  <input id="goal-weekly-target" type="number" min="1" max="7" value="4">
</div>
```

**Progress calculation:**
```js
function resolveHabitProgress(g) {
  if (g.goalType !== 'habit' || !g.weeklyTarget) return null;
  const now = new Date();
  const weekStart = fmtLocal(new Date(now.getFullYear(), now.getMonth(), now.getDate() - (now.getDay()+6)%7));
  const weekEnd = todayStr();
  const activeDates = Object.keys(sessions).filter(d => d >= weekStart && d <= weekEnd &&
    getSessionsForDate(d).some(s => s.exLogs && s.exLogs.length > 0)
  );
  const done = activeDates.length;
  const pct = Math.min(done / g.weeklyTarget * 100, 100);
  return { current: done, target: g.weeklyTarget, pct, label: `${done}/${g.weeklyTarget} this week` };
}
```

**On card:** Shows a mini progress bar + "3/4 this week" label. The deadline field shows "ongoing" instead of a date.

---

## Feature 9 — Notes / Check-ins Per Goal

A collapsible notes section on each goal card. Tapping "Add note" appends a timestamped entry to `g.notes[]`.

**Card structure (expanded):**
```html
<div class="goal-notes-section">
  <div class="goal-note-entry">
    <span class="goal-note-date">2026-06-28</span>
    <span class="goal-note-text">Hit 90kg today, form felt solid</span>
  </div>
  <div class="goal-add-note">
    <input class="modal-input" placeholder="Add a check-in note...">
    <button onclick="addGoalNote(i, value)">+</button>
  </div>
</div>
```

**Function:**
```js
function addGoalNote(idx, text) {
  if (!text.trim()) return;
  if (!goals[idx].notes) goals[idx].notes = [];
  goals[idx].notes.push({ date: todayStr(), text: text.trim() });
  save('goals', goals);
  renderGoals();
}
```

Notes section is collapsed by default (hidden), expanded by tapping a "Notes (2)" toggle link on the card. Uses existing `.day-exercises` / `.open` toggle pattern.

**CSS to add:**
```css
.goal-notes-section { display: none; border-top: 1px solid var(--border); padding: 10px 14px; }
.goal-notes-section.open { display: block; }
.goal-note-entry { display: flex; gap: 8px; margin-bottom: 6px; font-size: 11px; }
.goal-note-date  { color: var(--muted); flex-shrink: 0; }
.goal-note-text  { color: var(--text); }
.goal-notes-toggle { font-size: 10px; color: var(--muted); cursor: pointer; margin-top: 4px; display: inline-block; }
```

---

## Updated `addGoal()` Signature

After all features, `addGoal()` reads:
```js
function addGoal() {
  const text = document.getElementById('goal-input').value.trim();
  if (!text) return;
  goals.push({
    id:            Date.now(),
    text,
    deadline:      document.getElementById('goal-deadline').value || null,
    done:          false,
    doneAt:        null,
    category:      newGoalCategory,
    priority:      newGoalPriority,
    goalType:      newGoalType,          // 'binary' | 'measurable' | 'habit'
    target:        parseFloat(document.getElementById('goal-target')?.value) || null,
    unit:          newGoalUnit || null,
    linkExercise:  document.getElementById('goal-link-exercise')?.value.trim() || null,
    weeklyTarget:  parseInt(document.getElementById('goal-weekly-target')?.value) || null,
    notes:         [],
  });
  resetGoalForm();
  save('goals', goals);
  renderGoals();
}
```

---

## Updated `renderGoals()` Structure

```
renderGoals()
  ├── Apply goalsFilter (active / done / all)
  ├── Sort: overdue → priority → deadline
  └── For each goal:
       ├── goal-card (priority border colour)
       │    ├── done checkbox
       │    ├── category icon + goal name + PR badge (if measurable + done)
       │    ├── deadline label (urgency colour class)
       │    ├── progress bar + label  (measurable goals only)
       │    ├── habit bar + "X/Y this week"  (habit goals only)
       │    ├── "Notes (N)" toggle link  (if notes exist)
       │    ├── notes section (collapsed)
       │    └── delete button
```

---

## CSS Additions Summary

| Class | Purpose |
|---|---|
| `.deadline-ok/soon/urgent/today/overdue` | Urgency colour on deadline text |
| `.goal-card.priority-high/medium/low` | Left-border priority colour |
| `.goal-progress-bar-wrap` | Background track for measurable progress |
| `.goal-progress-bar` | Filled portion of progress track |
| `.goal-progress-label` | "Current: 72kg → Target: 100kg" text |
| `.goal-notes-section` | Collapsible notes area |
| `.goal-notes-section.open` | Expanded state |
| `.goal-note-entry` | Individual timestamped note row |
| `.goal-note-date` | Muted date prefix |
| `.goal-notes-toggle` | "Notes (2)" expand link |

All other UI (filter tabs, pills, chart-toggle buttons) reuses existing CSS classes.

---

## No Breaking Data Changes

All new fields are additive. Existing goals (`{ id, text, deadline, done }`) render correctly as `binary` goals with default styling. The only behavior change to existing goals is that `toggleGoal()` now also sets `doneAt`.
