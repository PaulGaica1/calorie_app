# CLAUDE.md

Guidance for working on this repo. Read this first.

## What this is
- **Habitrack** — a deliberately minimal personal calorie/meal tracker. The user logs
  food & drink (what, calories, time, a healthiness rating) and sees it charted + tabulated.
- **For:** one individual tracking their own intake — single-user, no accounts, no sharing.
- Emphasis is on *patterns*: calories per day, calories by part of day, meal healthiness,
  fasting windows, and a rolling daily average.
- Tiny and dependency-free: one HTML file, runs anywhere, data lives in the browser.
- Live at https://paulgaica1.github.io/calorie_app/ (GitHub Pages, served from `main`).

## Tech stack
- Plain **HTML + CSS + JavaScript**, all inline in a single `index.html`. No framework,
  no build step, no bundler, no `package.json`.
- **No dependencies** except two Google Fonts (Fredoka = display, Nunito = body) via `<link>`.
- **Persistence:** browser `localStorage` (key `calorie-entries`). No backend, no DB, no network.
- **Charts:** hand-rolled inline **SVG** (no chart library).
- **Hosting:** GitHub Pages from `main`. Repo `PaulGaica1/calorie_app`.

## Folder structure
```
calorie_app/
├── index.html   # the entire app: markup + <style> + <script> (single IIFE)
├── README.md
└── CLAUDE.md    # this file
```

## Install / run / test
- **Install:** nothing (no deps, no build).
- **Run locally:** open `index.html` in a browser, or serve it:
  `python3 -m http.server 8000` → http://localhost:8000
- **Publish:** merge to `main`; GitHub Pages redeploys in ~1–2 min. Auto-publish is ON —
  commit to the working branch, then merge to `main`.
- **Test:** no committed test suite; verify in a real browser. In this project's cloud
  dev environment, Playwright + Chromium are preinstalled; drive headless Chromium:
  ```
  node -e '(async()=>{const {chromium}=await import("/opt/node22/lib/node_modules/playwright/index.js");
  const b=await chromium.launch({executablePath:"/opt/pw-browsers/chromium"});
  const p=await b.newPage();await p.goto("file:///home/user/calorie_app/index.html");
  console.log(await p.title());await b.close();})()'
  ```
  Seed state via `localStorage["calorie-entries"]` (JSON array of entries) then reload.
  Always screenshot desktop (~880px) and mobile (~390px).

## Data model (do NOT rename fields)
Each entry in `localStorage`:
```
{ id, what, category, selfAssessment, prepMode, date, time, estimatedCalories, timestamp }
```
- `date`="YYYY-MM-DD", `time`="HH:MM" (local). `timestamp`="dateTtime:00", derived, sort-only.
- `category` (required): Breakfast | Lunch | Dinner | Beverage | Snack.
- `selfAssessment` (optional ordinal): "" (Not rated) | Nasty | Okay | Healthy-ish | Healthy.
- `prepMode` (optional): Home cooked | Fridge scraps | Delivery | Restaurant | Meal Prep | Pick-up | Office | Other.

### Weight entries (separate feature, separate store)
Weight tracking is its own self-contained feature — **never mix it into
`calorie-entries`**. Stored under `localStorage` key `weight-entries`:
```
{ id, weightKg, preBM, date, time, timestamp }
```
- `weightKg` (number, kg). `preBM` (optional): "Yes" | "No" | "".
- `date`/`time`/`timestamp` same conventions as calorie entries.

## Decisions we must NOT reverse
- **Single file, zero build, no dependencies, no backend.** Keep everything in `index.html`;
  data stays in `localStorage`.
- **Keep `migrateEntry()`** (in `loadEntries`) — it upgrades old saved data (old snack
  categories; "Healthy but high calorie"→"Healthy-ish"; "Misc"→unrated). Never drop it;
  real users have data.
- **Primary accent is teal** (`--accent`/`--accent-d`/`--accent-soft` = `#1b9c94` /
  `#10847e` / `#e2f4f2`), derived from the logo's turquoise; drives all calorie-side
  chrome (buttons, toggles, focus rings, time/day pickers, Today stat, default tag).
  **Weight stays blue** and the **health-rating scale stays green** (green = healthy
  is semantic) — those are not part of the accent and must not be rethemed with it.
- **Self-assessment is the 4-step scale** with fixed `SELF_COLORS` = red, yellow,
  light-green, dark-green (+ grey when unrated). Those colours also drive the charts.
- **Chart semantics:** bars are **stacked per meal in eating order** (never merge
  same-rating meals); **today** = lighter tones + no outline, **past** = base tones +
  outline; time-of-day view uses fixed **parts-of-day buckets** (Morning/Midday/Afternoon/
  Evening/Night); a dotted line marks **avg daily calories** over the last ≤10 logged days.
  The per-day chart **defaults to the most recent day** (scrolls horizontally, slide left
  for history) and keeps the **y-axis + legend pinned** while the bars scroll.
- **CSV export** keeps the `data_furball_lol_<date>_<time>.csv` name and 7-column layout;
  blank From/To = export everything. The **weight** log has its own export mirroring it,
  named `weight_furball_lol_<date>_<time>.csv` (4 cols: Weight (kg) / pre-BM / Date / Time).
- **Brand:** name "Habitrack"; logo is a rounded-square app icon (cream badge, mint
  circle, white checkmark) embedded as an inline PNG data URI in the `.brand .logo`
  `<img>`; teal primary accent matched to the logo; Fredoka+Nunito.
- **Time picker (both forms):** a custom **radial clock** composing a 24h `HH:MM` into a
  hidden input (`#time` / `#weight-time`), so downstream code is unchanged. AM/PM is
  **not stored**; times are **quarter-hour granularity** (loading snaps to the nearest
  quarter). **Collapsed by default**: the big live readout (e.g. `8:15 PM`) **is** the
  toggle — a light-gray rounded box that turns **green with white text when open** —
  above the quick buttons (**Now / 30 mins ago / 1h ago**). Expanded:
  **outer ring = hours**, **inner ring = minutes** (`:00/:15/:30/:45` only), **centre =
  AM/PM halves** (top AM / bottom PM). Drag around a ring to set it (value updates live);
  the AM/PM toggle **re-labels the hours to 24h** (AM `12,1–11` / PM `12,13–23`; `12` at
  top = midnight/noon). The weight form's copy is blue-themed.
- **Re-logging via the "What" autocomplete** copies a past entry's metadata (category,
  rating, prep, calories) but sets **date + time to now** — you're logging it again now,
  not reusing the old timestamp.
- **No zoom / always fit to screen:** viewport meta uses `maximum-scale=1,
  user-scalable=no`; `html` has `touch-action: manipulation` + `text-size-adjust: 100%`;
  iOS Safari pinch is blocked via `gesturestart/change/end` handlers. Deliberate, despite
  the accessibility trade-off. Date inputs are capped (~210px), not full width.

### Weight feature semantics (do NOT reverse)
- **Blue accent** for everything weight (vs. green for calories); the weight input
  card remaps `--accent*` to the blue set (including the time picker below).
- **Per-day chart = two stacked, scroll-synced blocks**: Calories (full height) on
  top, Weight (squished, `plotH` ~90) below. Both always show when they have data —
  **no mode toggle**. Section headers appear only when both blocks are present.
- **Weight chart uses a fixed ±2 kg vertical window** (widens only past a 4 kg span)
  and has **no axis labels**.
- **Weight hero figure** sits above the weight chart (under the "Weight" section
  header when both charts show): today's **7-day trailing average** in large blue,
  with the 7-day average **from a week ago + the kg change in parentheses**
  underneath in a lighter/muted line (delta tinted red rising / green falling per
  the moving-average colour language). Both averages reuse `movingAvgWeight()`
  (ignores blank days); hidden when there's no weigh-in in the last 7 days.
- **Weight colour language:** reported-weight **line = blue**; per-day weight **pills**
  = blue (pre-BM Yes/blank) or **purple** (pre-BM No); **7-day moving average** =
  light-red rising / light-green falling (chips + chart) and stronger red/green line
  segments. Moving average **ignores blank days**; MA chip label reads e.g. `72.3 kg`.
- **Weight log table** mirrors the entries table (newest-first, 5/page, edit/delete,
  shared day-filter). Editing a weight loads it into the weight form (Update/Cancel).

## Conventions & rules
- **Match the existing style:** ES5-flavoured vanilla JS (`var`, function declarations),
  everything inside the single IIFE in `index.html`. No frameworks, no new deps, no tooling.
- **One source of truth:** `render()` redraws chart + table + today stat after any data
  change; both charts go through the shared `drawStacked()` renderer; reuse existing helpers
  (`fmtNum`, `dateStr`, `timeToMin`, `escapeHtml`, `escapeCsv`).
- **Always escape** user text when injecting into HTML (`escapeHtml`) or CSV (`escapeCsv`).
- **Keep it minimal** — no accounts, no server, no nutrition database, no macros/targets.
  New features stay client-side and self-contained.
- **Test before publishing** (browser/Playwright + desktop & mobile screenshot), then
  commit and merge to `main`.
- **Git:** develop on `claude/calorie-tracking-app-0760bf`, merge to `main` to publish.
  End commit messages with the `Co-Authored-By: Claude …` trailer. Never commit the model
  identifier anywhere.
