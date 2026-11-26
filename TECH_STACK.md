# Sextus' Gym Workout Tracker — Tech Stack & Project Overview

## What It Does

A personal workout tracker web app. You paste in a training program using a custom plain-text format, and the app turns it into a structured weekly schedule you can check off in real time. It syncs across devices via Firebase and works offline.

**Core features:**
- Log sets, reps, weight, RPE, and status tags (completed / partial / skipped / fatigued / pain / easy / hard) per exercise
- Run-specific logging (distance, pace, cadence, HR) with a collapsible section
- Race Day and Ball Game session types for non-gym activity
- Calendar view with color-coded workout types
- Exercise progress chart (weight/reps over time per exercise)
- Streak and session history
- Goals checklist
- ChatGPT export — select sessions and copy formatted text for AI analysis
- Backdated logging — log any past date from the calendar
- Light / dark mode

**What it does not do:**
- No social features, no sharing, no coach-facing view
- No AI/LLM integration built-in (export is manual copy-paste)
- No native mobile app

---

## Stack

| Layer | Technology |
|---|---|
| Frontend | Vanilla HTML/CSS/JS — no framework, no bundler |
| Auth | Firebase Auth — Google Sign-In (popup) |
| Database | Firestore (cloud) + localStorage (offline fallback) |
| Hosting | Firebase Hosting |
| Fonts | Google Fonts — Barlow + Barlow Condensed |
| Dev dependency | firebase-tools (CLI for deploy only) |

No npm packages are loaded at runtime. The only runtime scripts are Firebase SDKs pulled from Google's CDN (`gstatic.com`).

---

## Project Structure

```
Health and Workout Trackers/
├── gym_tracker.html          # The entire app — HTML, CSS, and JS in one file (~1600 lines)
├── firebase.json             # Firebase Hosting config
├── .firebaserc               # Firebase project ID (workout-tracker-e7407)
├── package.json              # Only contains firebase-tools as a dev dep
├── package-lock.json
├── PROJECT_CONTEXT.md        # AI assistant context + architecture notes
├── IMPLEMENTATION_PLAN_23June2026.md  # Feature plan doc
├── IMPROVEMENTS.md           # Improvement backlog
├── SextusGymWorkoutTracker_user_guide.md  # User guide
├── chatgpt_menu_prompt.md    # Prompt template for ChatGPT export analysis
├── deployupdatecommand.txt   # Firebase deploy command reference
└── node_modules/             # firebase-tools only
```

Everything is in `gym_tracker.html`. There is no build step, no compilation, no bundler. Open the file in a browser and it runs.

---

## What Was Built

All of the following was custom-built — no UI library, no component framework:

- Custom program parser (`parseProgram()`) — reads plain-text training plans into a structured schedule
- Exercise logging modal — weight, reps, RPE, status tags, notes
- Run logging section — distance, pace, cadence, HR
- Race Day modal and Ball Game modal (new session types)
- Calendar view with color-coded workout types and day detail view
- Exercise progress chart (canvas-based, drawn with `drawChart()`)
- Streak and session statistics
- ChatGPT export selector — multi-select sessions, format to plain text
- Offline/online sync — localStorage loads first, Firestore syncs in background; conflict resolution keeps whichever side has more sessions
- One-time data migrations run on load (`migrateSessionDayIdx`, `migrateSessionsToArray`)
- Light/dark mode toggle with CSS variable swapping
- Backdated logging from calendar
- Custom/ad-hoc exercise logging (exercises not in the program)
- Session notes

---

## Deployment

**Status: Live**

- **URL:** https://workout-tracker-e7407.web.app
- **Hosting:** Firebase Hosting (static file host — serves `gym_tracker.html`)
- **Deploy command:** `firebase deploy`
- **Repo:** https://github.com/SextusEmpiricus1009/Workout_Tracker

No CI/CD pipeline. Deploy is manual via Firebase CLI from local machine.

---

## Auth

Firebase Auth with Google Sign-In. Users authenticate via Google popup. No email/password, no email link auth, no Cordova.

Firebase API keys are hardcoded in `gym_tracker.html` around line 435. This is intentional — for client-side Firebase apps, API keys are not secret. Security is enforced via Firestore security rules on the backend, not by hiding the key.

---

## Database

**Firestore** (Google's NoSQL cloud database, serverless).

Data is stored per authenticated user. The main data key is `sextus_data` — currently user-agnostic in naming (personal project origin). Firestore is used for cross-device sync only; localStorage is the primary read source on load.

**Key data shapes:**

```js
// Session (one entry per day, being migrated to array for multi-session support)
sessions["2026-06-23"] = [
  {
    entryType: 'workout',
    dayName: 'Lower A',
    dayType: 'Strength',
    exLogs: [{ name, weight, reps, rpe, notes, tags, dayIdx }],
    notes: ''
  }
]

// Program day
program[i] = {
  name, type, duration,
  exercises: [{ name, volume, tags, notes }]
}
```

---

## Backend

None. There is no server, no API, no serverless functions. Firebase Auth and Firestore are consumed directly from the browser via Firebase's client SDK. The app is fully client-side.
