# CLAUDE.md

Guidance for AI assistants working in this repository.

## Project overview

**MoDa Air Pollution Monitor** — a single-page dashboard that reads live and
historical air-quality data from [AirGradient](https://www.airgradient.com)
sensors via their public REST API and renders it as cards, time-series charts,
and a metric correlation matrix. Built for [moda.education](https://moda.education).

The entire application is **one static HTML file**. There is no backend, no
build step, no package manager, no test suite, and no CI. The browser is the
only runtime.

## Repository layout

```
index.html            The entire application (markup + CSS + JS, 778 lines)
MoDa_Logo_White.png   Logo, referenced by relative path from index.html
README.md             Two-line stub
```

That's the whole repo. If a change is needed, it almost certainly goes in
`index.html`. **Do not introduce a build system, framework, bundler, or
`node_modules` unless the user explicitly asks** — the single-file,
open-in-a-browser property is the point of this project.

## Running it

Open `index.html` in a browser, or serve the directory:

```bash
python3 -m http.server 8000   # then visit http://localhost:8000
```

Serving over HTTP is preferred over `file://`; the app's own error handler
suggests it when a fetch fails (`corsHint`, `index.html:768`).

On first load a modal asks for an AirGradient API token (AirGradient dashboard
→ *General Settings → Connectivity*). The token is stored **only** in
`localStorage` under the key `ag_token` and is sent as a `?token=` query
parameter. Never hard-code a token into `index.html` or commit one.

There is no automated verification. Validate changes by loading the page with a
real token and exercising the affected control.

## External dependencies (all CDN, all pinned except Tailwind)

| Dependency | Loaded at | Notes |
|---|---|---|
| Tailwind CSS (Play CDN) | `index.html:7` | Unversioned; all layout/color classes come from here |
| Chart.js 4.4.0 | `index.html:8` | UMD build, global `Chart` |
| chartjs-adapter-date-fns 3.0.0 | `index.html:9` | Required for the `type:'time'` x-axes |
| Nunito (Google Fonts) | `index.html:10` | Font family is set explicitly in Chart.js font options too |

The page does not work offline. Keep the pinned versions unless the user asks
to upgrade.

## Architecture

`index.html` is organized in three blocks, in order:

1. **`<style>` (`:11-47`)** — the handful of rules Tailwind can't express:
   the `--moda-yellow: #FFE400` custom property, `.active` states for the three
   button groups, the spinner, and the correlation-matrix table cells.
2. **Markup (`:49-223`)** — token modal (`:51`), nav (`:77`), title bar
   (`:105`), then `<main>`: error banner (`:116`), current-readings grid
   (`:119`), historical-chart section with its controls (`:133`), footer (`:218`).
3. **`<script>` (`:225-776`)** — all logic, plain ES2020, no modules. `init()`
   is called unconditionally at the bottom (`:775`); this works because the
   script tag sits at the end of `<body>`.

### Module map within the script

| Concern | Lines | Key symbols |
|---|---|---|
| Global state | `226-239` | `chart`, `locations`, `currentRecords`, `selectedMetric`, `selectedRange`, `viewMode`, `autoRefreshTimer`, `DAY_PALETTE` |
| API wrapper | `242-255` | `api(path, params)` |
| Token handling | `258-282` | `saveToken`, `showTokenModal`, `closeTokenModal` |
| Bootstrap / auto-refresh | `285-297` | `init`, `startAutoRefresh` (5-minute interval) |
| Current readings | `300-324` | `refreshData`, `normalizeArray` |
| Metric definitions | `327-350` | `METRICS`, `AQI_COLORS`, `AQI_LABELS`, `aqiLevel`, `aqiColor` |
| Location cards | `353-411` | `renderCards`, `coerce`, `fmt` |
| Location selection | `414-428` | `populateSelect`, `selectLocation` |
| Control handlers | `431-445` | `setMetric`, `setRange`, `setViewMode` |
| Field lookup | `448-464` | `ALT_KEYS`, `getValue` |
| Chart orchestration | `467-535` | `loadChart`, `showStats`, `commonScalesPlugins` |
| Chart renderers | `538-680` | `renderStandardChart`, `renderHourlyChart`, `renderDayOverlayChart` |
| Correlation matrix | `683-750` | `CORR_METRICS`, `pearsonR`, `corrCellStyle`, `renderCorrMatrix` |
| UI helpers | `753-772` | `setPlaceholder`, `showError`, `hideError`, `corsHint` |

### AirGradient API usage

Base URL: `https://api.airgradient.com/public/api/v1` (`:226`). Three endpoints:

- `GET /place` — place name for the nav subtitle; failure is swallowed (`:287`).
- `GET /locations/measures/current` — powers the card grid and the location
  `<select>`; polled every 5 minutes.
- `GET /locations/{locationId}/measures/past?from=…&to=…` — history for the
  chart; `from`/`to` are ISO strings derived from `selectedRange` in days (`:477-481`).

Every call goes through `api()`, which appends the token, throws on non-2xx
with a truncated response body, and returns parsed JSON.

### Data flow

```
init() → refreshData() → api('/locations/measures/current')
                       → normalizeArray() → renderCards() + populateSelect()

card click / select + Load → loadChart() → api('/locations/{id}/measures/past')
                                         → normalizeArray() → currentRecords
                                         → render{Standard|Hourly|DayOverlay}Chart()
                                         → renderCorrMatrix()
```

### View modes

`viewMode` selects the chart renderer in `loadChart` (`:487-489`):

- **`standard`** — every raw reading plotted against real timestamps. Point
  radius degrades with density (`:555`).
- **`hourly`** — readings bucketed by local calendar hour and averaged, plotted
  at the hour midpoint (`:565-602`).
- **`dayOverlay`** — one line per calendar day, all mapped onto a shared fake
  reference date (`Jan 2 2000`, `:624`) so the x-axis reads 00:00–23:59 and days
  can be compared directly. Series colors cycle through `DAY_PALETTE`.

### Correlation matrix

`renderCorrMatrix` (`:718`) builds a 5×5 Pearson-r table over `CORR_METRICS`
(particles, CO₂, temperature, TVOC, NOx) from the currently loaded records.
`pearsonR` drops pairs where either value is missing and returns `null` for
fewer than two valid pairs. Cell background interpolates neutral gray → red for
positive r and neutral gray → blue for negative r (`corrCellStyle`, `:708`).

Note this list is deliberately *not* `METRICS` — it excludes PM1/PM2.5/PM10 and
humidity. Changing one list does not change the other.

## Conventions

**Defensive API parsing.** The AirGradient payload shape and field names vary
between sensor models and firmware. Two mechanisms absorb that, and new code
should use them rather than reading fields directly:

- `normalizeArray(data)` (`:318`) accepts an array, `{data:[…]}`, `{locations:[…]}`,
  or a bare object, and always returns an array.
- `ALT_KEYS` + `getValue(rec, key)` (`:448-464`) try a list of candidate field
  names in priority order and return the first present value. **The
  `*_corrected` variant always comes first** — corrected readings win over raw
  ones. `renderCards` does the same thing inline with `??` chains (`:364-370`).

Missing values are `null`, never `0` or `NaN` — `coerce` (`:404`) enforces
this, `fmt` (`:408`) renders `null` as `—`, and card tiles for `null` metrics
are omitted entirely.

**DOM access.** Everything is `document.getElementById` against hard-coded IDs;
there are no data-binding helpers. Visibility is toggled with Tailwind's
`hidden` class via `classList.add/remove/toggle`. Active button state is a
`.active` class applied by matching `dataset.metric` / `dataset.range` /
`dataset.view` against the current global.

**Event wiring.** Inline `onclick="…"` attributes calling global functions. All
handler functions must therefore stay in the global scope — do not wrap the
script in an IIFE or convert it to a module without rewiring every handler.
The one exception is the document-level `Escape` listener (`:282`).

**HTML generation.** Template literals assigned to `innerHTML` (`renderCards`,
`populateSelect`, `renderCorrMatrix`). Values interpolated into these strings
come from the API and are not escaped — keep that in mind if adding fields that
could contain markup.

**Styling.** Tailwind utility classes inline; add a rule to the `<style>` block
only when a utility can't do the job. The palette is dark-first: `bg-gray-950`
page, `bg-gray-900` panels, `border-gray-800` edges, `text-gray-100` body,
`text-gray-400/600` for secondary text. MoDa yellow (`#FFE400`, the
`--moda-yellow` variable) is the primary accent and the `.moda-btn` background;
indigo marks the active view-mode button; `gray-700` marks the active range.

**Formatting.** 2-space indent, semicolons, `const`/`let`, single quotes,
compact one-line object literals for config, `// ── Section ───` banner comments
between concerns. Match the surrounding density — this file is deliberately terse.

## Common tasks

**Add a metric to the chart.** Four coordinated edits:
1. Add an entry to `METRICS` (`:327`) with `label`, `unit`, `color`, and
   `thresholds` (ascending AQI-style breakpoints; empty array = no AQI coloring).
2. Add the API field-name candidates to `ALT_KEYS` (`:448`), `*_corrected` first.
3. Add a `<button onclick="setMetric('key')" data-metric="key" class="metric-btn …">`
   to the metric group (`:145-155`).
4. Optionally add it to `CORR_METRICS` (`:683`) with a short label for the matrix.

**Add a metric tile to the location cards.** Extend the `coerce(...)` block in
`renderCards` (`:364-370`) and add a `tile(...)` call in the grid (`:391-397`).
Tiles self-hide when the value is `null`.

**Add a time range.** Add a `<button onclick="setRange(N)" data-range="N" class="range-btn …">`
(`:165-170`). `N` is days; `loadChart` handles the arithmetic. No JS change needed.

**Add a view mode.** Add a `.view-btn` with `data-view="name"`, then a branch in
`loadChart` (`:487-489`) and a `render…Chart(records)` function. Reuse
`commonScalesPlugins(m)` (`:509`) for consistent axis/tooltip styling and call
`showStats` plus `document.getElementById('chartPlaceholder').classList.add('hidden')`.

**Change AQI banding.** Edit `thresholds` in `METRICS` and, if the number of
bands changes, `AQI_COLORS`/`AQI_LABELS` (`:339-340`) — `aqiLevel` returns an
index into those arrays and falls through to the last band.

## Known quirks

- **The token modal is not hidden on load when a token already exists.** The
  modal div has no `hidden` class in the markup (`:52`), and `init()` returns
  early only in the *no token* case — the token-present path never hides it
  (`:285-293`). Returning users must press `Escape` or click Cancel to dismiss
  it. Worth fixing if you're touching the token flow.
- `corrCellStyle` (`:713`) picks `NEU` for both branches of the first
  destructuring — intentional in effect (both directions start from neutral),
  but the ternary is redundant.
- The auto-refresh interval reloads the cards only; the chart never refreshes on
  its own and must be re-loaded with the Load button.
- `currentRecords` (`:229`) is assigned in `loadChart` but not read anywhere
  else; it's a hook for future features.

## Git workflow

- Default branch is `main`; the working branch for assistant changes is
  specified per task. Push with `git push -u origin <branch>` and open a draft PR.
- History is short and the repo has no linters or hooks — commit `index.html`
  directly with a descriptive message.
- Never commit an API token, and check that `localStorage` fixtures or debug
  values haven't been left in the file before committing.
