# CLAUDE.md

Guidance for working on this repo. Read this first.

## What this is
- **HabiCat** — a deliberately minimal personal calorie/meal tracker. The user logs
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

## Decisions we must NOT reverse
- **Single file, zero build, no dependencies, no backend.** Keep everything in `index.html`;
  data stays in `localStorage`.
- **Keep `migrateEntry()`** (in `loadEntries`) — it upgrades old saved data (old snack
  categories; "Healthy but high calorie"→"Healthy-ish"; "Misc"→unrated). Never drop it;
  real users have data.
- **Self-assessment is the 4-step scale** with fixed `SELF_COLORS` = red, yellow,
  light-green, dark-green (+ grey when unrated). Those colours also drive the charts.
- **Chart semantics:** bars are **stacked per meal in eating order** (never merge
  same-rating meals); **today** = lighter tones + no outline, **past** = base tones +
  outline; time-of-day view uses fixed **parts-of-day buckets** (Morning/Midday/Afternoon/
  Evening/Night); a dotted line marks **avg daily calories** over the last ≤10 logged days.
- **CSV export** keeps the `data_furball_lol_<date>_<time>.csv` name and 7-column layout;
  blank From/To = export everything.
- **Brand:** name "HabiCat"; pillowy white badge + green cat-head SVG logo; Fredoka+Nunito.

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
