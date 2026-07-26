# HANDOFF

Working session handoff for **HabiCat**. Read `CLAUDE.md` first for the durable
project rules; this file is the volatile "where we are right now" note.

- **Working branch:** `claude/read-claude-md-1ynzvi` (merged to `main` per-change via
  squash PRs; `main` auto-deploys to GitHub Pages).
- **Last updated:** 2026-07-26

## 1) Completed this session
All merged to `main` (PR numbers in parentheses):

- **Per-day chart defaults to the most recent days**; scrolls horizontally, slide
  left for history; y-axis + legend stay pinned while bars scroll (#2, #3).
- **Weight tracking added.** New "Weight" mode on the Add-entry card via a
  Calories/Weight toggle: weight (kg, text), pre-BM? Yes/No, date + time. Stored
  under its own `localStorage` key `weight-entries`, separate from calories (#4).
- **Weight card is blue-themed**; weight records render under each day in the
  per-day chart (#5), and show for weight-only days too (#6).
- **Chip-based time picker** replaced the native time input in both forms:
  12 hour chips (4×3) · AM/PM · :00/:15/:30/:45 → composes 24h `HH:MM` (#7).
- **Weight log table** under the Entries table — same features (newest-first,
  5/page, edit/delete, day-filter, CSV export `weight_furball_lol_*.csv`);
  columns Weight (kg) / pre-BM / Date / Time (#8).
- **7-day moving-average chip** under each day's weight pill (ignores blank days;
  light red rising / light green falling) (#8).
- **Weight line charts**: blue reported-weight line + red/green moving-average
  line, then reworked so Calories (full) and Weight (squished) stack as two
  separate synced blocks with a fixed **±2 kg** weight scale (#9, #10).
- **Removed the Calories/Weight/Both toggle** — both charts now always show,
  each only when it has data (#11).
- **Weight pills coloured by pre-BM**: No = purple, Yes/blank = blue; line stays
  blue (#12).

## 2) In progress
Nothing mid-flight. The last change (pre-BM pill colours, #12) is complete,
merged, and deployed. Working tree is clean apart from this handoff commit.

## 3) Known bugs / issues / rough edges
- **Quarter-hour snapping:** the chip time picker only offers :00/:15/:30/:45, so
  editing an older entry saved at an odd minute (e.g. `12:41`) snaps to the
  nearest quarter (`12:45`) on load/save. Expected, but worth remembering.
- **Weight-only day shows an empty calorie bar:** the per-day chart uses the union
  of calorie + weight dates, so a day with a weigh-in but no food shows a
  zero-height calorie column. Cosmetic.
- **Weight chart stays squished even when it's the only chart** (calorie-only-less
  view). Fine, but could be given more height when standalone.
- **pre-BM left blank renders blue** (same as Yes) — intentional default; no
  distinct colour for "unspecified".
- No automated test suite committed; verification is via Playwright/Chromium ad hoc
  (see CLAUDE.md) plus desktop/mobile screenshots.

## 4) Next tasks (priority order)
- [ ] Decide weight-only-day handling: hide the empty calorie column, or leave it.
- [ ] Give the weight chart more height when it is the only chart on screen.
- [ ] Consider a distinct colour/legend entry for blank (unspecified) pre-BM.
- [ ] Add a lightweight committed test/verification checklist for the weight
      features (chart modes, table edit/delete, CSV) to speed future sessions.
- [ ] Optional: show the ±2 kg span subtly on the weight chart (it has no labels),
      so the calm scale is legible.

## 5) Decisions made today
These are now also recorded in `CLAUDE.md`:
- Weight tracking is a **second, self-contained feature**: its own `localStorage`
  key `weight-entries`, entry shape `{ id, weightKg, preBM, date, time, timestamp }`,
  never mixed into `calorie-entries`. Blue accent (vs. green for calories).
- Time is entered via a **chip picker** (hours 1–12 · AM/PM · quarter-hours) that
  composes a 24h `HH:MM`; AM/PM is not stored; times are quarter-hour granularity.
- Per-day chart shows **Calories (full height) and Weight (squished) as two
  stacked, scroll-synced blocks**, always both when data exists (no mode toggle).
- Weight chart uses a **fixed ±2 kg** vertical window (widens only past a 4 kg
  span) and **no axis labels**.
- Colour language: reported-weight **line = blue**; weight **pills** = blue
  (pre-BM Yes/blank) or **purple** (pre-BM No); **moving-average** = light-red
  rising / light-green falling (chips) and stronger red/green line segments,
  ignoring blank days.
- Weight CSV export mirrors the calorie one but is named
  **`weight_furball_lol_<date>_<time>.csv`** (calorie export keeps `data_furball_lol_`).
