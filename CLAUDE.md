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
  **One opt-in exception:** the official Anthropic SDK (`@anthropic-ai/sdk@0.131.0`, pinned)
  is `import()`ed from jsDelivr only when the user taps ✨ Estimate with their own API key
  saved — see "AI estimates" below. Without a key, nothing is downloaded.
- **Persistence:** browser `localStorage` (key `calorie-entries`). No backend, no DB. The only
  network call the app ever makes is the opt-in ✨ Estimate request to Anthropic.
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

### Habit state (app bookkeeping, not user data)
Stored under `localStorage` key `habit-state`, separate from both data stores:
```
{ milestones: { <key>: <ISO date first earned> }, recapDismissed: "<week key>", onboarded: bool,
  fastSeen: "<last-meal timestamp>|<hours nodded>", remindersSet: bool, remindDismissed: bool }
```
Read through `normalizeHabitState()` so missing/unknown fields fall back to defaults.

### AI settings (opt-in, device-only)
`localStorage` key `ai-settings` = `{ apiKey }` — the user's own Anthropic API key. Never
written to backups, never restored from them; removed entirely when the user taps Remove.

### Backup file (JSON)
`habitrack_backup_<date>_<time>.json` =
`{ app: "habitrack", version: 1, exportedAt, calorieEntries, weightEntries, habitState }`.
Restore validates every row (bad rows are counted and skipped, calorie rows go
through `migrateEntry()`), asks for confirmation with before/after counts, replaces
the data, and offers a 10 s Undo.

## Decisions we must NOT reverse
- **Habit guardrail:** the habit features reward *consistency* (showing up, logging),
  **never eating less**. No red "failure" states, no guilt copy, forgiving streaks;
  milestones are about logging, never about calorie numbers.
- **Undo, not confirm:** saves and deletes (food + weight) act immediately and show a
  5 s toast with **Undo** (`showToast` + `restoreRecord` / `discardRecord` /
  `deleteRecord`). Only a full **restore** asks for confirmation first, because it
  replaces everything.
- **Four tabs, hash-routed** (`#today` · `#log` · `#trends` · `#history`; `showTab` / `goTab`):
  **Today** = Today panel, weekly recap, "Log again" quick-log card, consistency heatmap,
  reminders nudge · **Log** = the Add entry card (calories + weight) · **Trends** = the
  charts · **History** = Entries + Weight log tables and Your data. Every feature lives in
  exactly one panel; moving them never changes their logic. Back goes to the previous tab, a
  reload keeps the tab, a fresh launch opens Today (`history.scrollRestoration = "manual"`).
  The chart is (re)drawn when Trends is shown, because it sizes/scrolls from its visible
  width. Editing from History opens Log and **returns to History** after Save/Cancel
  (`openLogTab(returnTo)` / `finishEdit()`); a heatmap tap opens History filtered; a chart-bar
  tap offers "View" in a toast.
- **Tab bar:** fixed bottom bar on mobile (≤560px) whose raised teal **+** is the Log tab —
  tapping it from another tab focuses the active form's first field (this replaced the
  earlier floating "+"). On desktop it is a sticky pill under the header. On mobile it hides
  while a text field is focused (`body.kb-open`) and `scroll-padding-bottom` keeps
  scrolled-to elements above it, so it can never cover the field in use. Toasts sit above it.
  **Tapping + animates:** the button squishes and springs back with a quarter turn
  (`replayAnim(.tab-plus, "pop")`, on every tap) and the Log panel rises into place
  (`panel-rise`, ~0.32 s) whenever Log opens — other tabs switch instantly. The bar waits
  for the bounce (~0.43 s) before stepping aside for the keyboard. Reduced motion: no
  bounce, the panel only fades. `replayAnim()` removes its class on animationend/cancel,
  so a later display change never re-runs an animation.
- **Quick log** (top of the calorie form **and** the Today tab's "Log again" card): up to 6 one-tap chips from `canonicalByWhat()`,
  ranked by logs in the last 30 days (+1 per log at the current part of day), ties →
  most recent. Tap = `addEntry()` **now** (time snapped to the quarter hour) with the
  item's usual category/rating/prep/calories; **✎** = Log tab + `applySuggestion()` (never
  over an open edit). Hidden with no history; the form's copy also hides while editing.
- **Smart category default:** a fresh form pre-selects the category for the time of day
  (`PART_CATEGORY` over `DAY_PARTS`: Breakfast / Lunch / Snack / Dinner / Snack) and keeps
  following the clock (`makeClockPicker`'s `onChange`) until the user picks one, a
  suggestion is applied, or an entry is being edited.
- **Toasts never block what's beneath:** the toast body is `pointer-events: none`; only
  its action button takes taps.
- **Save celebration:** every *new* log (food teal, weight blue) pops a logo-like check
  (`celebrate()`) + `navigator.vibrate` where supported; updates don't celebrate. The
  Today number counts up to its new value. Both respect `prefers-reduced-motion`.
- **Today panel** (top card): SVG ring of today's calories vs **your usual day**
  (`usualDailyCalories()` = mean of the last ≤10 logged days *before* today — a reference,
  never a goal; the ring is simply full at/over it, no overflow or red), items logged,
  healthy share of *rated* items (`dayStats()`), and a **fasting timer** since the last
  item logged at or before now (`lastMealInfo()`; hidden after 48h). Fasting gets a quiet
  toast once per fast at **12h and 16h only** — deliberately nothing beyond, so the app
  never rewards ever-longer fasts. Seen marks are kept in `habitState.fastSeen`.
- **Logging streak** (`streakInfo()`, header badge → Milestones sheet): consecutive days
  with **any** food or weight log. Forgiving and stateless (computed from the data, so
  back-filling repairs it): one missed day in any 7 is bridged as a **rest day**; a run ends
  only on two missed days in a row or a second missed day within a week; today never
  counts as missed. While today is unlogged the badge goes calm/muted ("pending", never
  red) and the Today panel nudges "Log anything today to keep your N-day streak".
- **Consistency heatmap:** last 12 weeks (columns = Sun–Sat weeks, this week last). Every
  logged day is filled — not-so-healthy days still count as showing up — with green
  deepening by healthy share (`l0` soft teal = nothing rated → `l4`); blue dot = weigh-in.
  Tapping a logged day sets `filterDate` (same path as a chart bar, cells use
  `data-hdate`) and opens the History tab.
- **Milestones** (`MILESTONES`): first log, first weigh-in, streaks 3/7/14/30/60/100 (by
  best streak), 100/250/500/1000 entries, 10/30 weigh-ins — only ever about logging.
  Earned once (date in `habitState.milestones`), celebrated once with confetti + a toast
  that **queues behind** the action's own Undo toast (`queueToast`); qualifying history
  earns them in one go. Confetti is skipped with reduced motion.
- **Insight of the day** (foot of the Today card): `INSIGHTS` generators over your own data
  (prep-mode healthy split, share of calories after 5pm, longest overnight fast this week,
  healthiest weekday, healthy-day run, 7-day weight vs a month ago, days logged, go-to item,
  usual first bite). Each returns a **neutral** sentence or `null` without enough data; the
  pick rotates daily (stable within a day), › shows the next. Rendered via `textContent`
  (entry names are user text).
- **Weekly recap** (`renderRecap`): on **Mondays** last full Mon–Sun week vs the one before,
  shown until dismissed (`habitState.recapDismissed` = that week's Monday); any day via
  "Your week ›" = this week so far vs the same days last week. Days logged and healthy
  share get a teal "good" delta when up; calories, fasting and weight deltas stay neutral.
- **Shared day helpers:** `entriesByDate()` + `fastWindowMin(byDate, date)` (overnight
  window) are top-level and reused by the chart's FW captions, insights and recap.
- **Calendar reminders** (no server, so no push): the Reminders sheet (Today nudge card once
  2+ days are logged, or History → Your data → "Calendar reminders") downloads
  `habitrack_reminders.ics` with daily `RRULE:FREQ=DAILY` events + `VALARM`s in **floating
  local time**, with stable UIDs (re-importing updates instead of duplicating). Defaults are
  your median Breakfast/Lunch/Dinner times over 60 days **+30 min** and your median weigh-in
  time (fallbacks 08:30 / 13:00 / 19:30 / 07:30); every time is editable. Lines are folded
  at 75 octets with CRLF endings (RFC 5545). The nudge goes away once set up or on "Not now".
- **First-run welcome:** 3 short cards for a brand-new, empty app; closing it any way marks
  `habitState.onboarded`. The last card offers Start logging (Log tab, focused) or reminders.
  Empty states point to the next step ("Log your first meal" on Today).
- **Sheets** (Milestones, Reminders, Welcome) share `openSheet(html, onClose)` /
  `closeSheet()`: one at a time; ✕, backdrop tap or Escape close them.
- **✨ AI estimates (opt-in, bring-your-own key):** with a key saved in History → Your data,
  the Log form shows **✨ Estimate** beside "What". Tapping it lazily imports the pinned SDK
  (`AI_SDK_URL`), creates `new Anthropic({ apiKey, dangerouslyAllowBrowser: true,
  timeout: 30000, maxRetries: 1 })`, and calls `client.beta.messages.create` with
  `model: "claude-opus-5-5"`, `effort: "low"`, `fallbacks: "default"` (beta
  `server-side-fallback-2026-07-01`) and a **json_schema structured output**
  (`ESTIMATE_SCHEMA`: calories, category, rating, prep — enums = the app's own vocab — and a
  ≤12-word note). The prompt sends only the typed text, the form's local time, and up to 8
  of the user's own past items sharing words with it (`similarEntries`). The system prompt
  requires the note to be **neutral** (no judgement, no diet advice — the habit guardrail).
  The reply is validated (`parseEstimate`; calories clamped 1–5000) and **only fills the
  form** — never saves — with an "✨ AI estimate · note" line and an Undo toast; it is
  discarded if "What" changed meanwhile. Refusal / max_tokens / bad JSON / 401 / 429 /
  other API errors / no connection each give a plain toast and leave the form untouched.
  Model output is rendered via `textContent` only. Bumping the SDK version: pick a stable
  release at least two weeks old and re-run the AI tests.
- **Toasts shrink to fit** (`min-width: 0`): long messages end in "…", Undo stays visible.
- **`.hidden` is `!important`** so a component rule that sets `display` can never un-hide
  something the code hid.
- **Live tick:** every 30 s (and on returning to the app / day rollover) the panel
  redraws, and an **untouched** form's date/time moves on to now (`autoStamp`), so a log
  after the app sat open never lands on a stale time or yesterday. A date/time the user
  set themselves is never overwritten.
- **Install as an app:** `<head>` carries `theme-color`, `apple-mobile-web-app-*` and an
  **opaque** 180px `apple-touch-icon` data URI (iOS renders transparent icon corners
  black). No service worker or manifest file — that would break the single-file rule.
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
- **Pillowy 3D surfaces:** section `.card`s and the header `.today-stat` badge use a
  warm white→cream gradient (`--card-grad`) + a layered "raised pad" shadow
  (`--shadow-3d`: bright top rim, soft warm bottom edge, soft ambient drop),
  echoing the icon's puffy 3D badge. Keep it subtle — don't let it turn heavy.
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
  **The pinch blocker must keep skipping interactive controls** (`input, textarea,
  select, button, [contenteditable]`) — suppressing the default over a field stops it
  focusing and the on-screen keyboard never opens. Likewise the radial clock only takes
  a pointer capture while a ring is actually being dragged, releases it on
  pointerup/pointercancel, and has a window-level safety net that clears the drag state:
  a stuck capture would swallow taps meant for other controls.

### Weight feature semantics (do NOT reverse)
- **Blue accent** for everything weight (vs. green for calories); the weight input
  card remaps `--accent*` to the blue set (including the time picker below).
- **Per-day chart = two stacked, scroll-synced blocks**: Calories (full height) on
  top, Weight (squished, `plotH` ~90) below. Both always show when they have data —
  **no mode toggle**. Section headers appear only when both blocks are present.
- **Weight chart uses a fixed ±2 kg vertical window** (widens only past a 4 kg
  span) with a **3-tick kg y-axis (hi/mid/lo) + faint gridlines**, pinned on the
  left and mirroring the calorie chart's axis. The scale is hoisted in
  `drawStacked()` so the pinned axis and the lines share one `weightWy()` mapping.
- **Weight hero figure** sits above the weight chart (under the "Weight" section
  header when both charts show): today's **7-day trailing average** in large blue,
  then an **interactive comparison line**: `Up|Down <Δ> from <base-date> <7-day avg
  on that date>` (delta tinted red rising / green falling), or `Same as <date>
  <avg>` when unchanged, or `No data on <date>` when the base has no average. The
  **base date is a blue calendar popup** (`.wb-btn` / `.wb-pop`, reusing the `.dp-*`
  calendar markup, remapped to the weight blue) and **defaults to 7 days ago**;
  only past dates are selectable, cells use `data-wbdate` (not `data-date`) so the
  day-bar filter ignores them. Averages reuse `movingAvgWeight()` (ignores blank
  days); the whole hero is hidden when there's no weigh-in in the last 7 days.
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
