# Front end (`index.html`)

The whole UI is one file, about 5,100 lines: HTML shell, CSS, then a single `<script>`. There's no framework and no build step. That keeps hosting trivial (GitHub Pages) and lets anyone open the file and read it top to bottom.

## File layout

| Lines (approx.) | Part |
|---|---|
| 1–11 | `<head>`: fonts, supabase-js 2.117.3 (UMD from jsDelivr) |
| 12–1070 | CSS: tokens, layout, components, print styles, the "streamlined" layer, page-specific blocks (certificate, charts, map…) |
| 1071–1077 | Body shell: `#bar` (top bar), `#app` (page), `#actionsSlot` (sticky action bar) |
| 1078–5095 | JavaScript, in sections marked `// ── Section name ──` |

Search for `// ──` to jump between sections. In order:

| Section | Holds |
|---|---|
| Connection | Supabase URL and publishable key, `sb` client |
| State & helpers | `S` (app state), `STATUS`, `DETAILS`, `FOOTPRINT`, `esc`, `num`, scoring (`peachScore`), `fetchAll`, `toast`, `fail`, `friendly`, `notice` |
| Form versions, Settings | `qsFor`, `useVersion`, `appVersion`, `applySettings`, `levelFor`, `finalScore`, due-date helpers |
| Theme | `loadTheme`, `applyTheme`, `brandHTML`, font pairings |
| Boot & routing | `boot`, `start`, `route`, `renderBar` |
| Staff settings | certification levels, review target, form versions (shown on the Questions page) |
| Certificate | `certificateHTML` (shared renderer), `renderCertificate`, `openSendDialog`, `renderVerify` (public page) |
| Certificate designer | `renderCertDesign` |
| Geography | `geoLine`, `openGeoDialog`, `searchPlaces`, `locateProduction`, `loadMapKit`, `renderMapChart` |
| Leadership report | `renderReport`, `REPORT_BLOCKS`, report customization |
| Excel import | `parsePeachWorkbook`, `wireImport` (SheetJS loaded on demand) |
| Application page pieces | certification header, banners, reviewer flags, delegates ("Who's answering"), activity and comments |
| Notifications bell | `loadNotes`, `subscribeNotes` (Realtime), `drawBell` |
| Auth | sign in, sign up, password reset and recovery |
| Application list, Board | `renderList`, the kanban board with drag-and-drop |
| Application detail | `openApp`, `loadRelevantContacts` (the "Answered by" suggestions list only contacts already on the application, added by its owner, named on the owner's other applications, or at its studio; one datalist per department, leaving out people from other departments), `drawApp`, `drawActions`, `saveApp`, evidence files. The page is two columns: the form in `.app-main`, and a sticky `.app-side` sidebar holding the score tally, Application tools (Attachments: every file on the application grouped by department, each linked to its action, with Download all as a .zip built in the browser by JSZip 3.10.1 from cdnjs; Import from Excel, certificate, Review, Who's answering, Delete) and an "On this page" list that jumps to departments and shows each one's progress. On phones the sidebar stacks above the form and the tools collapse |
| Dashboard | `renderDashboard`, `buildDashHTML` (built-in widgets), `drawDashboard` |
| Charts | `DIMS`, `measures`, `chartData`, `renderChart`, `svgBar`, `htmlHbar`, `svgLine`, `svgDonut`, `tableView` |
| Layout, Cross-filters | `dashWidgets`, `resolveLayout`, `saveDashPrefs`, `xfSet`, `wireXf`, `wireBrush` |
| Chart builder | `openChartBuilder` |
| Compare (Analysis) | `renderCompare`, `CMP_FACETS`, `practiceScore` |
| Staff: people | `renderPeople` (Admin: users, contacts, companies) |
| Design, My account, Staff: questions | `renderDesign`, `renderAccount`, `renderQuestions` |

## State

Everything lives on one object, `S`:

| Key | What |
|---|---|
| `S.session`, `S.me` | Supabase session; the signed-in person's `profiles` row |
| `S.allQs`, `S.allDepts` | Every active question and department, across versions |
| `S.qs`, `S.depts` | The version currently in view (set by `useVersion`) |
| `S.formCurrent`, `S.cert`, `S.reviewDays`, `S.certCfg`, `S.theme` | From `app_settings` |
| `S.people` | Profiles, loaded for reviewers and staff (names in lists) |
| `S.current` | The open application: `{ app, resp, confirm }` |
| `S.dash` | Dashboard data: `allApps`, `apps` (after cross-filters), `by` (answers by app and stage), `respBy` |
| `S.dashPrefs`, `S.sharedCharts`, `S.dashDefault` | Dashboard layout and charts |
| `S.dashFilter` | Cross-filters `{ dim: [keys] }` |

Data is fetched fresh on each page render, with no client cache. Realtime is used only for the notifications bell and the open application's comments.

## Routing

The page uses hash routing. `route()` runs on every `hashchange`:

| Hash | Page | Who |
|---|---|---|
| `#applications` (default), `#board` | List / kanban | everyone |
| `#new`, `#app-<id>` | Application form | everyone (RLS decides) |
| `#cert-<id>` | Certificate | anyone who can see the application |
| `#verify-<token>` | Public certificate | anyone, no sign-in |
| `#dashboard` | Dashboard (default for staff) | reviewer, staff |
| `#analysis`, `#analysis-<qid>` | Analysis (formerly `#compare`) | reviewer, staff |
| `#report` | Leadership report | reviewer, staff |
| `#questions`, `#admin`, `#contacts`, `#companies`, `#design`, `#design-certificate` | Settings pages | staff |
| `#account` | My account | everyone |
| `#map`, `#people`, `#compare` | Old links; redirected or aliased | |

Role checks in `route()` only decide which pages are offered. The real enforcement is RLS.

## Rendering conventions

- Pages render by setting `$app.innerHTML` from template strings, then wiring event listeners.
- **Always pass user data through `esc()`** before putting it in HTML, including attribute values. A `data-tip` value is HTML shown in the tooltip, so anything inside it that would break the attribute must be escaped as well. A stray `"` there once hid a whole map.
- Errors go through `fail(e)`, which logs the error and shows a toast with `friendly(e)`. Errors from edge functions should be unwrapped with `fnError(error, data)` so the function's own message is shown.
- Saving is explicit: forms don't save on change, apart from dashboard layout, which saves as you arrange.
- Accessibility: real buttons and links, `aria-pressed` on toggles, `role="status"` for messages, focus kept after re-renders where it matters (`refocus`), keyboard alternatives to dragging (arrow buttons on the dashboard, a stage menu on board cards).

## Charts and the dashboard

A chart is a **spec**, a small JSON object saved in `dashboard_prefs.charts`, in `shared_charts.spec`, or in a report layout:

```js
{ id, title, type,   // "bar" | "hbar" | "line" | "donut" | "map" | "number" | "table"
  measure,           // key from measures(): "count", "score", "co2t", "m:<metric>", "cat:<category>", ...
  agg,               // "avg" | "median" | "sum" | "min" | "max"
  stage,             // "wrap" | "prep" (for PEACH measures)
  dim, split,        // keys of DIMS (group by / split by); maps use "country" or "region"
  size, height,      // "third" | "half" | "twothirds" | "full"; "s" | "m" | "l" | "xl"
  filters }          // { status: [...], type: [...], ... } (empty means everything)
```

- `chartData(spec, crossFilter)` turns a spec plus `S.dash.allApps` into groups and series. `renderChart(el, spec, …)` draws them.
- Series colors come from the CSS tokens `--s1`…`--s6`, a palette checked for color-blind separation, assigned in a fixed order so a series keeps its color when filters change.
- **Built-in widgets** (`DASH_BUILTIN`: key numbers, pipeline, categories…) are HTML from `buildDashHTML`. The map widget (`geo`) is a preset spec from `geoPreset()`.
- **Layout** is a list `[{ id, size, height, hidden }]`. Precedence: the person's own layout, then the staff default (`app_settings.dashboard_default`), then `DASH_DEFAULT`. New widgets are added to the end of saved layouts.
- **Cross-filtering:** clicking a mark calls `xfSet(dim, keys)`, and dragging across marks brushes several. Every widget then shows only matching applications, except the chart grouped by the filtered dimension, which keeps all its groups and highlights the selection.

### Maps

`renderMapChart` draws a choropleth with d3-geo and topojson-client. Both libraries are loaded on first use by `loadMapKit()`, along with outlines from world-atlas@2.0.2 (countries) or us-atlas@3.0.1 (states).

- Countries are matched by ISO code (`ISO_NUM`, alpha-2 to the atlas's numeric ids), falling back to the name (`nameKey`).
- States are matched by name.
- Values are shaded in 5 steps; dots show individual productions.
- Canadian provinces have no outline data, so Canada is shaded as one country.

## Theming

- **CSS tokens** are defined on `:root`: colors, fonts, radius, chart palette. Dark values are defined under `prefers-color-scheme: dark` and `[data-theme="dark"]`.
- **Staff overrides** (Design page) are stored in `app_settings.theme`. `applyTheme()` writes a `<style>` element that redefines the tokens, deriving the dark-theme colors from the chosen ones.
- **Components** use tokens only. Don't add literal colors.

## External libraries

All pinned to exact versions:

| Library | Version | Loaded |
|---|---|---|
| supabase-js | 2.117.3 | always (jsDelivr) |
| SheetJS (xlsx) | 0.18.5 | on Excel import (cdnjs) |
| JSZip | 3.10.1 | on Download all attachments (cdnjs) |
| d3-array, d3-geo, topojson-client | 3.2.4, 3.1.1, 3.1.0 | on first map (jsDelivr) |
| world-atlas, us-atlas | 2.0.2, 3.0.1 | on first map (jsDelivr) |

When upgrading one, change the URL in one place and test the pages that use it.
