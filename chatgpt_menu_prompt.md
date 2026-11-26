# Prompt for ChatGPT — generate a menu my FORM tracker can parse

Paste everything below into ChatGPT (attach this file or copy the whole thing). It tells ChatGPT to output your weekly plan in the exact plain-text format my tracker's parser expects, so I can paste the result straight into the Program tab.

---

You are writing a weekly training plan for me. Output **only** plain text in the exact format defined below — no markdown, no headings, no bullet points, no commentary before or after. I paste your output directly into an app that parses it literally, so formatting rules are strict.

## Structure

- One block per training day. A block starts with a `DAY:` line, then a blank line, then one exercise per line.
- Put a blank line between day blocks.
- Use a weekday at most once.

## DAY line

```
DAY: <Weekday> <Label> | <Type> | <Duration>
```

- **The first word after `DAY:` must be a weekday**: Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, or Sunday. This is required — a block whose name doesn't start with a weekday is discarded.
- `<Label>` is a short free-text descriptor (e.g. "Lower + Push", "Easy Run", "Recovery"). Optional but recommended.
- `<Type>` must be exactly one of: **Strength**, **Aerobic**, **Hybrid**, **Recovery**. (Use Aerobic for run/cardio-focused days — those show pace/distance in the calendar; all others list exercise names.)
- `<Duration>` is optional, written as `Nmin` (e.g. `60min`).

## Exercise lines

```
Exercise Name | field | field | field ...
```

At least one field after the name. Fields can be in any order. Recognized field formats (must match exactly, **no spaces inside a token**):

- **Sets×reps:** `NxN` — e.g. `3x8`. Use a single rep target, not a range (write `3x10`, not `3x8-12`).
- **Time:** `Nmin` — e.g. `5min`, `45min`.
- **Distance:** `Nkm` or `N-Nkm` — e.g. `5km`, `4-5km`.
- **Weight:** `Nkg` — e.g. `45kg`. Always include `kg`; a bare number is treated as a note.
- **RPE:** `rpe N` — e.g. `rpe 8`.
- **Tag** (gives a colored label) — use one of exactly these: `lower`, `upper`, `core`, `aerobic`, `recovery`, `cooldown`, `mobility`, `rehab`, `agility`, `conditioning`, `coordination`.
- **Note:** any other free text is kept as a note (use this for cues like "slow eccentric", or for timed-interval detail like "8 rounds 20s on / 40s off").

### Avoid these (they won't parse as intended)
- Rep ranges (`3x8-12`) — pick one number.
- Seconds tokens (`20sec`, `15s`) as a sets/reps field — put interval timing in a note instead.
- Spaces inside a token (`3 x 8`, `5 km`, `45 kg`).
- Markdown, numbering, or extra prose.

## Template to follow

```
DAY: Monday Lower + Push | Strength | 60min

Warmup Walk | 8min | recovery
Leg Press | 3x10 | lower
Romanian Deadlift | 3x8 | lower
Chest Press | 3x10 | upper
Lat Pulldown | 3x10 | upper
Cable Row | 3x12 | upper
Calf Raise | 3x12 | lower | slow eccentric
Bike Intervals | conditioning | 6 rounds 15s hard / 45s easy
Cooldown Walk | 5min | cooldown

DAY: Wednesday Easy Run | Aerobic | 50min

Easy Outdoor Run | 4-5km | aerobic | conversational pace
Walking Cooldown | 5min | recovery
```

Now produce my plan in this exact format.
