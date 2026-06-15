# Workout Tracker — Project Context

## What This Is
A single-file web app (`gym_tracker.html`) for tracking workouts. No framework, no build step — just HTML, CSS, and vanilla JS in one file. Deployed via Firebase Hosting.

**Live URL:** https://workout-tracker-e7407.web.app (check `.firebaserc` for project ID)  
**Repo:** https://github.com/SextusEmpiricus1009/Workout_Tracker

---

## Tech Stack
- **Frontend:** Single HTML file — all CSS and JS inline, no bundler
- **Auth:** Firebase Auth (Google Sign-In)
- **Database:** Firestore (cloud sync) + `localStorage` (offline fallback)
- **Hosting:** Firebase Hosting
- **Only dependency:** `firebase-tools` (dev, for deployment only)

---

## App Screens
| Screen | ID | What it does |
|---|---|---|
| Today | `screen-today` | Active workout view — check off exercises, log sets/reps/weight/RPE |
| Progress | `screen-progress` | Calendar, session history, exercise progress chart, streak stats |
| Goals | `screen-goals` | Simple milestone checklist |
| Program | `screen-program` | Paste and parse a workout program in custom text format |

---

## Key Architecture Notes

**Data flow:** `localStorage` is loaded first (instant), then Firestore syncs in the background. On conflict, whichever has more sessions wins (rough heuristic). This means the app works offline.

**Program format:** Plain text, parsed by `parseProgram()`. Format is:
```
DAY: <Name> | <Type> | <Duration>
<Exercise Name> | <volume> | <muscle tag> | <optional notes>
```
Types: Strength, Aerobic, Hybrid, Recovery, Other  
Tags: lower, upper, core, aerobic, recovery, cooldown, mobility, rehab, agility, conditioning, coordination

**Session data structure:**
```js
sessions[dateStr] = {
  dayName, dayType,
  exLogs: [{ name, weight, reps, rpe, notes, tags, dayIdx }]
}
```

**State globals:** `program[]`, `checks{}`, `logs{}`, `goals[]`, `sessions{}`, `selectedDay`

**Save pattern:** `save(key, val)` writes to `localStorage` immediately and syncs to Firestore in the background (non-blocking). `saveAll()` writes everything at once.

---

## Notable Features
- **ChatGPT export** — select sessions and export formatted text for AI analysis (`exportForChatGPT()`)
- **Run logging** — separate fields for distance/pace when exercise is a run type (`updateRunSection()`)
- **Backdated logging** — can log to any past date via calendar day detail view
- **Custom/ad-hoc exercises** — log exercises not in the program for a given day
- **Status tags** — per-exercise tags (completed, partial, skipped, fatigued, pain, easy, hard)
- **Exercise progress chart** — tracks weight/reps over time per exercise (`drawChart()`)
- **One-time migration** — `migrateSessionDayIdx()` backfills old sessions that predate the `dayIdx` field

---

## New Machine Setup

```bash
# 1. Clone
git clone https://github.com/SextusEmpiricus1009/Workout_Tracker.git
cd Workout_Tracker

# 2. Install Firebase CLI
npm install -g firebase-tools

# 3. Authenticate Firebase (browser popup)
firebase login

# 4. Restore dev dependencies
npm install

# 5. Deploy (when ready)
firebase deploy
```

> No build step needed. Open `gym_tracker.html` directly in a browser for local testing.

---

## AI Assistant Context (Cowork)
- Cowork session history is **local only** — it does not carry over to a new machine or new session
- Paste this file into your first message when starting a new Cowork session on a different machine so Claude has full context
- The entire app is in `gym_tracker.html` (~1566 lines as of last update) — share that file when asking for code changes

---

## Current State / Last Known Work (update this when switching machines)
- Initial commit only — all features built in one shot before first push
- No open bugs documented
- `.gitignore` added after initial commit — run `git rm -r --cached node_modules .firebase .idea` then commit to stop tracking those folders

---

## Things Worth Knowing
- Firebase config (API keys) is hardcoded in `gym_tracker.html` around line 435 — this is intentional for a client-side Firebase app (keys are not secret, security is handled by Firestore rules)
- The default program in the file (`DEFAULT_PROGRAM`) is a real training plan (gym + agility + running focus)
- `migrateSessionDayIdx()` runs once on load — safe to leave in
