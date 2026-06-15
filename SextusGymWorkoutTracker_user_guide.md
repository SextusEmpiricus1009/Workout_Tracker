# Sextus' Gym Workout Tracker — User Guide

Sextus' Gym Workout Tracker is a personal gym + running tracker. You give it a weekly training menu, check off and log your workouts, and it builds a history, calendar, and progress charts. This guide walks a brand-new user through everything.

---

## Getting started

1. Open the app and tap **Sign in with Google**. Your data is tied to your Google account.
2. Your data is saved on your device instantly and synced to the cloud in the background, so it follows you across devices and survives a browser refresh. It also works offline — changes sync next time you're online.

There are four tabs along the bottom: **Today**, **Progress**, **Goals**, and **Program**.

---

## Step 1 — Load your weekly menu (Program tab)

The **Program** tab is where you set up the week's plan.

The easiest path:

1. Tap **Copy ChatGPT Prompt**. This copies a ready-made prompt to your clipboard.
2. Paste it into ChatGPT and let it generate your plan.
3. Copy ChatGPT's reply, paste it into the box on the Program tab, and tap **Parse Program**.

You can also write the menu by hand. The format is strict because the app reads it literally:

```
DAY: Monday Lower + Push | Strength | 60min

Warmup Walk | 8min | recovery
Leg Press | 3x10 | 81kg | lower
Romanian Deadlift | 3x8 | 40kg | lower
Cooldown Walk | 5min | cooldown
```

Rules that matter:

- **Every day must start with a weekday** (Monday–Sunday). The app places each day into its weekday slot by reading that first word. A day that doesn't start with a weekday is skipped.
- The part after the weekday becomes the day's label (e.g. "Lower + Push").
- **Type** must be one of: Strength, Aerobic, Hybrid, Recovery. Use **Aerobic** for run/cardio days.
- Each exercise is `Name | field | field | ...`. Recognized fields (no spaces inside a token): sets×reps `3x8`, time `5min`, distance `5km` or `4-5km`, weight `45kg`, effort `rpe 8`, and tags (lower, upper, core, aerobic, recovery, cooldown, mobility, rehab, agility, conditioning, coordination). Anything else becomes a free-text note.

After parsing, the app reports how many days it loaded and warns about any it couldn't place.

---

## Step 2 — Do your workout (Today tab)

The **Today** tab shows all seven weekdays as tabs across the top. Today's weekday is selected by default and outlined. Tap any weekday to view that day.

For the selected day you'll see its exercises. For each one you can:

- Tap the **circle** to check it off as done.
- Tap **LOG** to record what you actually did.
- Tap the **×** to remove that exercise from the plan.

Tap the **pencil (✎)** on the day's header to edit that day's label, type, and duration. (The weekday itself is fixed — Monday stays Monday.)

### Logging an exercise

The LOG screen lets you record:

- **Weight (kg)** — leave blank for bodyweight or cardio.
- **Reps / Volume** — reps, or something like `5km` or `30min`.
- **RPE (1–10)** — how hard it felt.
- **Running metrics** — pace, cadence, duration, average HR. This section opens automatically on Aerobic and Hybrid days, and you can open it manually any time.
- **Notes** — free text.
- **Status tags** — completed, partial, skipped, fatigued, pain, easy, hard, drop set, or your own custom tag.

It pre-fills with your last entry for that exercise, so progressive overload is just a quick tweak.

### Custom / substitute exercises

Tap **Custom exercise** to log something that isn't in the day's plan (a substitute machine, an extra set, an unplanned run). It appears under "Added this day," where you can **EDIT** or delete it.

### Session notes

The box at the top of the Today tab is for how the session felt overall — energy, soreness, recovery.

> **Tip:** If you select a weekday other than today and log into it, the entry is saved to that weekday's date in the current week.

---

## Logging a workout you did on an earlier day (Progress → Calendar)

If you don't log on the same day you trained, use the calendar to backdate correctly:

1. Go to the **Progress** tab.
2. In the calendar, tap the day you actually worked out.
3. The day detail opens. Tap **Add exercise** to log into that date, or **EDIT** / delete existing entries, and add session notes.

The day is filed under the correct date automatically — no mismatches.

---

## The weekly reset

At the start of each new week (Monday), the Today tab clears the previous week's exercises and shows a banner prompting you to paste the new menu. Your weekday tabs stay; only the plan empties out.

**Your logged history is never deleted** — it lives in the calendar and history permanently. Only the current week's plan resets.

---

## Tracking progress (Progress tab)

- **Stats row** — current streak, workouts this week, this month, and all-time total.
- **Calendar** — each logged day is colored by type. Run days show distance · duration · pace; other days list the exercise names; days with nothing logged stay plain. Tap any day to open its detail.
- **Session History** — a scrollable list of past sessions, filterable by Week / Month / Year / All. Tap a session to expand it.
- **Exercise Progress** — pick an exercise to see a weight-over-time chart plus its full log history. (Needs at least two logged sessions with weight to draw a trend.)

---

## Goals tab

Add a milestone (e.g. "Run 10km non-stop") with an optional target date. Check it off when you hit it, or delete it. Simple and persistent.

---

## Sharing your data with ChatGPT

On the **Program** tab, **Export for ChatGPT** lets you pick which sessions to include (last 7 days, 2 weeks, month, or hand-pick), then copies a clean summary of those sessions, your exercise progression, and your current program. Paste it into ChatGPT to ask for feedback or a new plan.

---

## Backup & restore

Also on the Program tab:

- **Export JSON** — downloads a full backup file of everything (program, logs, sessions, goals).
- **Import JSON** — restores from a backup file.

It's worth exporting a backup occasionally, especially before pasting a brand-new program.

---

## Quick reference

| I want to… | Where |
|---|---|
| Load this week's plan | Program → Copy ChatGPT Prompt → paste reply → Parse |
| Check off / log today's workout | Today → circle / LOG |
| Edit a day's name or type | Today → pencil (✎) |
| Delete an exercise from the plan | Today → × next to the exercise |
| Log a workout from an earlier day | Progress → tap the date → Add exercise |
| Fix or delete a logged entry | Today or Progress → EDIT / × |
| See trends and streaks | Progress |
| Set a goal | Goals |
| Back up everything | Program → Export JSON |

---

*Sextus' Gym Workout Tracker keeps your plan and your history separate: the plan is this week's intention, the history is the permanent record of what you actually did.*
