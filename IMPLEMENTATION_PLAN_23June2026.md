# Gym Tracker — Feature Implementation Plan

## Features
1. **Race Day entries** — log race/event results using existing run metric fields
2. **Ball Games entries** — log sport sessions (casual, practice, competition) with duration, HR, and free-text result
3. **Multi-session per day** — sessions array restructure so multiple entry types can coexist on the same date
4. **Light / Dark mode** — theme toggle persisted to localStorage

---

## 1. Data Structure Migration

### sessions array restructure

**Before:**
```js
sessions["2026-06-23"] = { dayName, dayType, exLogs, notes }
```

**After:**
```js
sessions["2026-06-23"] = [
  { entryType:'workout', dayName, dayType, exLogs, notes },
  { entryType:'raceDay', ... },
  { entryType:'ballGame', ... }
]
```

### New migration function — `migrateSessionsToArray()`
- Runs once on load alongside existing migrations
- Wraps every existing single-object session in `[...]`
- Tags it `entryType: 'workout'`
- Safe to run multiple times (checks if already an array)

### Race Day session shape
```js
{
  entryType: 'raceDay',
  dayName: '<event name>',   // e.g. "Bukit Timah 10K"
  dayType: 'Race Day',
  distance: '',              // free text, e.g. "10km"
  pace: '',                  // min/km
  cadence: '',               // spm
  duration: '',              // minutes
  hr: '',                    // avg bpm
  notes: ''
}
```

### Ball Game session shape
```js
{
  entryType: 'ballGame',
  dayName: '<sport>',        // free text, e.g. "Basketball"
  dayType: 'Ball Game',
  sessionType: 'casual' | 'practice' | 'competition',
  duration: '',              // minutes
  hr: '',                    // avg bpm
  notes: '',
  result: ''                 // free text, competition only — e.g. "Won 21-18"
}
```

---

## 2. New Helper Functions

| Function | Purpose |
|---|---|
| `getSessionsForDate(dateStr)` | Returns `sessions[dateStr]` as array; handles legacy single-object format transparently |
| `getWorkoutSession(dateStr)` | Returns the `entryType:'workout'` entry from the array (used by Today screen) |
| `openRaceDayModal(dateStr)` | Opens race day modal for a given date (today or past) |
| `saveRaceDay(dateStr)` | Reads modal fields, pushes race day entry to `sessions[dateStr]` array |
| `openBallGameModal(dateStr)` | Opens ball game modal for a given date (today or past) |
| `saveBallGame(dateStr)` | Reads modal fields, pushes ball game entry to `sessions[dateStr]` array |
| `toggleTheme()` | Flips `data-theme` attribute on `<html>`, saves preference to localStorage |
| `initTheme()` | Reads saved theme preference on load and applies it |

---

## 3. CSS Changes

### Theme system
Replace hardcoded dark values with swappable CSS variables:

```css
:root {
  --bg: #0d0d0f;
  --surface: #141416;
  --surface2: #1c1c1f;
  --border: #2a2a2e;
  --text: #f0f0f0;
  --muted: #666;
  /* accent colors unchanged in both modes */
}

[data-theme="light"] {
  --bg: #f5f5f7;
  --surface: #ffffff;
  --surface2: #ebebed;
  --border: #d0d0d4;
  --text: #1a1a1a;
  --muted: #888;
}
```

### New badge classes
```css
.day-type-badge.race-day   { /* orange-red */ }
.day-type-badge.ball-game  { /* purple */ }
.cal-cell.type-race-day    { /* calendar dot color */ }
.cal-cell.type-ball-game   { /* calendar dot color */ }
```

---

## 4. New HTML Modals

### Race Day modal (`race-modal`)
Fields:
- Event name (text input)
- Distance (text, e.g. "10km")
- Pace (min/km)
- Cadence (spm)
- Duration (min)
- Avg HR (bpm)
- Notes

### Ball Game modal (`ballgame-modal`)
Fields:
- Sport (free text input)
- Session type — 3-button toggle: **Casual** / **Practice** / **Competition**
- Duration (min)
- Avg HR (bpm)
- Notes
- Result (free text) — visible only when Competition is selected

### Theme toggle button
- Small ☀️/🌙 icon button in the Today screen header (top right)
- Always visible, no extra screen needed

---

## 5. UI Entry Points

### Today screen
- Below the day card: `+ Log Race Day` and `+ Log Ball Game` buttons
- Pass `dateForSelectedDay()` as the dateStr (follows the active weekday tab, so backdating works)

### Calendar day detail
- Alongside existing `+ Add exercise` button: `+ Log Race Day` and `+ Log Ball Game`
- Pass the calendar's `selectedCalendarDate` as dateStr (works for any past date)

### Backdating behaviour
All modal open functions accept a `dateStr` parameter and write directly to `sessions[dateStr]`. Past dates work identically to today — no special handling needed.

---

## 6. Existing Code Updates

| Function | Change |
|---|---|
| `renderToday()` | Use `getWorkoutSession()` for exLogs; add Race Day + Ball Game entry buttons |
| `saveSessionNotes()` | Find/create workout entry in array; don't touch other entry types |
| `saveCalendarNotes()` | Same as above |
| `saveLog()` | Push to workout entry in array |
| `deleteLoggedEntry()` | Index into array correctly; remove entry if empty; remove date key if array empty |
| `renderCalendar()` | Use `getSessionsForDate()` to iterate all entries for color + summary |
| `getSessionSummary()` | Handle multiple entries per day (workout + race + ball game) |
| `renderCalendarDayDetail()` | Render distinct sections per entry type; add entry buttons |
| `renderHistory()` | Different card layout for race day and ball game vs. workout |
| `getFilteredDates()` | Any non-empty entry in the array passes the filter |
| `exportForChatGPT()` | Include race day and ball game entries in export text |
| `renderExportSelector()` | Show entry type label in each row |
| `importData()` | Run `migrateSessionsToArray()` after import |
| `migrateSessionDayIdx()` | Update to work with array structure |

---

## 7. Implementation Order

1. **Theme system** — CSS variables + `toggleTheme()` + `initTheme()` + toggle button HTML; isolated, no dependencies
2. **Migration + helpers** — `migrateSessionsToArray()`, `getSessionsForDate()`, `getWorkoutSession()`
3. **New CSS badges** — race day and ball game colors
4. **New modals HTML** — race day modal, ball game modal
5. **New save functions** — `saveRaceDay()`, `saveBallGame()`
6. **Update all reads** — renderCalendar, getSessionSummary, renderCalendarDayDetail, renderHistory, getFilteredDates, renderToday
7. **Update all writes** — saveLog, deleteLoggedEntry, saveSessionNotes, saveCalendarNotes
8. **UI entry point buttons** — Today screen + calendar day detail
9. **Export updates** — exportForChatGPT, renderExportSelector
10. **Final: run all migrations on load** — wire `migrateSessionsToArray()` into `loadAll()`

---

## Notes

- No new nav tabs or screens — everything slots into existing UI
- `getSessionsForDate()` handles legacy data transparently so old backups still load correctly
- Import/export JSON backup format remains backward compatible (migration runs on import)
- Theme preference stored under key `sextus_theme` in localStorage (separate from main data key)
