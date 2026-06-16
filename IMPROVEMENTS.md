# Sextus' Gym Workout Tracker — Improvement Roadmap

A prioritized plan based on a code review (Codex appraisal) cross-checked against the current `gym_tracker.html`. Ordered by value ÷ effort. Nothing here is implemented yet except the items noted under "Done."

---

## Done (this session)
- Weekday-based program model (7 fixed Mon–Sun slots).
- Correct date/day attribution for backdated logging + export reads.
- Delete controls for planned and logged exercises (Today + Calendar).
- Calendar run-detection by logged metrics; empty days revert to plain; empty sessions dropped.
- Removed duplicate `migrateSessionDayIdx`; fixed apostrophe escaping in exercise names.
- **Export regression fix:** "CURRENT PROGRAM" section now uses the day's display name (was printing `DAY: undefined` after the weekday refactor) and skips empty rest days.

---

## Phase 0 — Correctness (small, highest priority)

**Date local/UTC normalization.** Many flows still use `new Date().toISOString().slice(0,10)` to mean "today." In UTC+8, any evening after 16:00 local rolls to the next calendar day, so a workout logged tonight can land on tomorrow's date, break the streak, and show on the wrong calendar cell.
- Add a single `todayStr()` helper using local date parts (reuse `fmtLocal`).
- Replace every `toISOString().slice(0,10)` with it: `getStats`, `getFilteredDates`, `renderCalendar` today highlight, and the `activeLogDate || ...` fallbacks in `openModal` / `openCustomModal` / `openEditModal` / `saveLog`.
- Re-verify streak and week-boundary math after the swap.

Effort: low. Risk: low. Impact: high (fixes wrong-day/wrong-week bugs you'd actually hit).

---

## Phase 1 — Security & data integrity

**Firestore security rules (highest security item).** Confirm the rules restrict each user to their own document, e.g.:
```
match /users/{uid} {
  allow read, write: if request.auth != null && request.auth.uid == uid;
}
```
Commit `firestore.rules` and `firestore.indexes.json` to the repo so the backend config is reproducible, not just set in the console.

**HTML escaping for user-rendered values.** The app renders exercise names, notes, labels, goals, tags, and session notes via template strings into `innerHTML`. Severity is moderate (single-user, self-XSS only; main vector is importing a hostile backup), but the fix is cheap and also prevents a stray `<` in a note from breaking the layout.
- Add `escapeHtml(s)` and apply it everywhere user data is interpolated into `innerHTML`.

**Harden `importData`.** Currently only checks `data.program && data.logs`. Add: type/shape validation per field, reject malformed or oversized input, and confirm before overwriting existing data.

Effort: low–medium. Risk: low.

---

## Phase 2 — Sync robustness

`loadAll()` adopts the cloud copy only when it has *more* sessions, so edits that don't change session count (goals, this week's program) can be silently overwritten by a stale device, and ties lose.
- Write a document-level `updatedAt` on every `saveAll()`.
- On load, merge by freshness (last-write-wins on `updatedAt`) instead of session count.
- Keep it simple — no per-entity vector clocks for a 1–2 device single user. Optionally add per-section timestamps later if conflicts persist.

Effort: medium. Risk: medium (touches the load/merge path — back up before changing).

---

## Phase 3 — Quality gate / tests

No automated tests today; `npm test` is a placeholder.
- Extract pure logic into testable form: `parseProgram` / `parseField`, date utils, `getSessionSummary`, and the sync-merge function.
- Add a minimal Node test runner (no heavy framework); wire `npm test`.
- Smoke tests: program parsing (incl. weekday mapping + unmatched days), JSON import/export round-trip, ChatGPT export generation, date/streak math.

Effort: medium. Risk: low. This is the real prerequisite for any future refactor.

---

## Phase 4 — Architecture (optional, only if graduating past prototype)

The whole app is one ~1,700-line HTML file (markup + CSS + state + parsing + persistence + auth + charting). That's the biggest long-term maintainability risk, but also the highest-effort, highest-regression-risk change with the least direct user benefit — and it slightly erodes the trivial single-file deploy.
- Pragmatic first step: split into `index.html` + `styles.css` + `app.js`.
- Only if it keeps growing: modularize into `storage.js`, `parser.js`, `render.js`, `state.js`.
- Decide explicitly: stay a clever single-file app, or invest in structure. Don't split just because it looks cleaner — do it because tests + team + scope justify it.

---

## Deliberately deprioritized
- Full multi-file decomposition before there are tests to catch regressions.
- Treating XSS as the top fix — it's self-XSS in a single-user app; correctness bugs (dates, export) produce wrong data right now and rank higher.
