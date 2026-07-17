---
name: dashboard-verify
description: Verify the Edmonton rainfall dashboard (dashboard/index.html) end-to-end after any edit to it — or when the user asks whether the dashboard numbers are right, why it shows something odd, or to test it. Serves it locally, cross-checks every KPI against independent live SoQL queries, and exercises the trend-buffer and feed-failure paths.
---

# Dashboard Verification

Never claim the dashboard works from reading its code — drive it and cross-check its numbers
against the source independently.

## Serve & load

`file://` is blocked in the Browser pane. Use the configured launch server
(`.claude/launch.json` → name `dashboard`, python http.server on port 8643), then open
`http://localhost:8643/index.html`. If a screenshot times out, verify via `get_page_text` /
`javascript_tool` — the DOM checks below are the authoritative pass/fail anyway.

## Checks (all must pass)

1. **Row/marker parity** — table rows = map markers = station count returned by the API
   (normally 23).
2. **KPI cross-check** — recompute independently with curl and compare:
   ```
   .../x4rm-mppc.json?$select=max(_24h_total)                     → Peak KPI (compare numerically — API max is a string max, so recompute client-side if totals span digit lengths)
   .../x4rm-mppc.json?$select=storm_rating,count(*)&$group=storm_rating → band counts, stations ≥4
   ```
3. **Freshness stamp** — page's "Data as of" equals newest `:updated_at`
   (`$$exclude_system_fields=false`) converted to America/Edmonton.
4. **Trend buffer** — key `edm_rain_buffer_v3`, shape `[{t:<ms>, stamp:<ms|null>,
   st:{RG17:[mm,rating],…}}]`. Hybrid sampling: a point when the batch `:updated_at`
   advances PLUS a steady point every ≥10 min (`t` = sample wall-clock) — so a 24h
   time-true line accumulates while the page stays open. Page refreshes inside the 10-min
   window must NOT add points (the old per-refresh regression showed everything as
   "steady +0.0 mm/5m"; flat segments during quiet feeds are correct and expected).
   Inject a synthetic point dated ~40 min back with offset totals, `memBuf=null; load()`,
   confirm rising/falling arrows (Action-now column + priorities panel) and the
   "N samples over X min" note.
   **Remove the synthetic snapshot afterwards** — it poisons real trends.
5. **Failure path** — stub `window.fetch` to reject, call `load()`, confirm the failbar shows
   "DATA UNAVAILABLE … do not assume no rain" and the stamp reads "feed unreachable". Restore
   fetch and reload.
6. **Popup chart** — with a multi-point buffer injected (step 4), `markerLayer.eachLayer` →
   `openPopup()` on a marker must show an SVG line with one path point per buffered reading and
   the "session data … not official history" caption; with a single-snapshot buffer it must show
   the "builds while the dashboard stays open" message instead of an empty plot.
7. **District choropleth (yellow→red ramp)** — 15 named planning districts from `avbh-ga6n`
   (Central, Scona, Jasper Place…), each filled by its worst gauge: 0 `#fee79a`, 1 `#fdd561`,
   2 `#fdb63f`, 3 `#f78f33`, 4 `#e95f2b`, 5 `#d03b3b`, ≥6 `#8a1616`; no-gauge district = gray
   `#c8c7c0` ("no visibility"). Gauges are uniform dark dots (`#1a1a19`); dot popup includes
   the district name + 24h chart; district popup lists its stations. Station→district is
   client-side point-in-polygon; RG60 is ~4 km outside city limits → "near Southeast"
   (nearest-district fallback, not an error).
7j. **Citywide alert bar is concise with a simple, stable cell color** (user request
   2026-07-15): background stays plain `var(--surface-1)` in EVERY state — the band color
   appears only as a 5px LEFT accent strip (`ab.style.borderLeft`) plus the big level number;
   no full-cell wash, no band-colored full border (the old `band.wash` background was removed
   at user request — don't reintroduce it). Content is two short lines: "icon Citywide: BAND —
   action" and "Worst RGxx · district · Shape — N ≥ Monitor, ≈ K km" (the verbatim feed quote
   was dropped HERE for brevity — it still lives in the table's "Feed says" column and popups,
   which is what satisfies the quote-verbatim rule; blind-spot warning line still appends in
   red when gauges go silent).
7a. **Shift brief panel** — a 6-cell grid directly under the alert bar, above Priorities,
   consolidating the decision picture: NOW (band + worst gauge + district, from render()),
   CHANGING (▲ rising / ▼ easing station lists from the session buffer; "no trend yet" under
   2 samples; "steady" when nothing moves), INCOMING next-24h (Open-Meteo brief incl. ⚠
   extreme flag), CREW WORK WINDOW next-6h, ECCC ALERTS (mirrors the KPI tile), BLIND SPOTS
   (red station list or "none — all N gauges reporting"). Every cell has an explicit
   failure/empty state; none may sit at "loading…" after the feeds resolve.
7b. **Priorities panel** — ordered duty list derived from state: ESCALATE (≥6) → BARRICADE (5)
   → DISPATCH (4) → PRE-POSITION (rising 3s) → REASSESS (falling ≥4) → VERIFY (blind spots)
   → ROUTE AROUND (emergency disruptions count). Each item is a READABLE SENTENCE (user
   request 2026-07-15): verb chip, then a capitalized imperative ending with a period —
   e.g. "BARRICADE — Close underpasses and low areas in <b>Northwest</b> (RG40, RG31) and
   <b>Jasper Place</b> (RG33)." — districts joined naturally via `joinNice` (commas + "and",
   no " · " separators, no trailing dash fragments like "— enable the closures map layer").
   Empty state reads "ROUTINE MONITORING — No station is at Monitor or above…". Don't regress
   to the old fragment style.
7c. **Sortable columns** — clicking any header sorts (numeric columns start descending);
   arrow indicator on the active header; blind rows stay pinned on top regardless of sort.
   There is NO "Updated" column (freshness lives in the header stamp only) and NO trend
   column — the merged "Rating · action now" column stacks the band chip over the action
   text, plus a rising/falling arrow when the session buffer has one.
7d. **Forecast** — the KPI tile + fcHead line show the Open-Meteo next-24h brief (total mm,
   peak hour, max gusts, ⚠ extreme-weather flag at ≥25 mm total / ≥10 mm/h / ≥70 km/h gusts).
   The 7-day panel is rendered from ECCC's own CORS-open citypage API
   (`api.weather.gc.ca/collections/citypageweather-realtime/items/ab-50`) into `.fcday`
   cards — day/night periods pair into one card (each half keeps its OWN icon + condition
   text: day on top, a `.fcnight` section with night icon / "night N°C" / condition below a
   divider), a leading "Tonight" gets its own card, official weathericons GIFs, "Forecast
   issued … MT" line, 30-min cache. Each card shows its calendar date (`.dd`, "Jul 13" —
   card i = today + i days, Edmonton time). Below the grid, `.fcsum` renders every period's
   full ECCC textSummary as aligned `.row` grid rows (period name + date + chips in a left
   column, text right; night rows indented/muted under their day). NO iframe — this replaced
   the cropped weather.gc.ca embed (fragile to alert banners); don't reintroduce one. The
   same payload's `warnings` drives the "Weather alerts — Edmonton" KPI tile ("None" when
   clear; red count + event names when active). On API failure both degrade to explicit
   "unavailable" text; nothing blocks the rest of the dashboard.
8. **Filters** — the filter bar lives in its OWN slim panel at the top of the Operations tab,
   above both the maps column and the table. District / Station / Rating / Feed-says selects
   filter the table rows AND gauge dots; the District filter ALSO hides every context-layer
   marker outside the district (pumps, closures, 311 flood reports, cameras — via
   `applyContextFilter()`: point-in-polygon district cached on each marker, hidden by CSS
   display, re-applied on overlayadd/loader completion). Ward fills and citywide KPIs must
   NOT change; Clear restores everything; selections survive an auto-refresh.
7e. **Crew work window** — `#fcCrew` under the forecast headline: next-6h rain phrase
   (dry / easing ≈ HH / rain returns ≈ HH), gusts (high-wind caution at ≥60 km/h), temp range,
   daylight-remaining from Open-Meteo `daily=sunrise,sunset`; empty on forecast failure.
7f. **District crew context** — a district popup appends "Crew context:" with pump count,
   closure count (emergency bolded), 311 flood-report count (14d), and nearest camera with ~km;
   computed at popup-open time via point-in-polygon on `pumpsList`/`closuresList`/`flood311List`.
7g. **311 flood layer (q7ua-agfg)** — window is the last 14 DAYS (label "(14d)" everywhere:
   KPI tile, layer toggle, popups, crew context); enabling it shows one marker per
   neighbourhood whose badge is ALWAYS the exact report count — including "1" (the old "F"
   glyph for singletons was removed at user request; every badge must be numeric and match
   `flood311List[].n`); red = underpass flooding, amber otherwise. Popup lists
   date · description · status and MUST carry the "neighbourhood centroid … updates daily,
   not real-time" caveat. Fetched once per session.
7i. **KPI tile text is CONCISE and tile backgrounds are static** (user requests 2026-07-15,
   twice tightened): tile titles are bare — "Weather alerts" (no "— Edmonton"), "Next 24 h"
   (no "— citywide forecast"), "Road closures" (no "in effect"). Each `.tile .n` line is one
   short fragment with NO "see X layer" pointers, NO source attributions ("(ECCC)",
   "Open-Meteo model"), and NO station codes in the attention tile: worst gauge = "station ·
   rating N band · district" (the single station code here is the gauge's identity and
   stays); attention = STACKED per-band counts ("1 Escalate<br>3 Barricade<br>4 Dispatch"
   plus "N blind spot(s)" line when present); 311 = "N open · N underpass flooding · N
   neighbourhoods · daily feed"; pumps = "in N hot district(s)"; closures tile leads with
   the EMERGENCY count as the big value ("11 EMERGENCY", red when >0; "0" otherwise) with
   "N total in effect" demoted to the line below — emergencies are what the supervisor acts
   on, the total is context. Do not re-inflate any of these, and NEVER change a tile's
   background color dynamically — `.tile` stays `var(--surface-1)` in every state (value
   TEXT color may still flip red for alarms; the background may not).
7h. **Lightning stand-down note** — toggling the lightning layer shows/hides `#lightningNote`
   ("stand down open catch-basin / pump work"); hidden by default.
8b. **Live traffic cameras (City of Edmonton)** — enabling the layer shows 58 `▣` markers
   (city intersections, source edmontontrafficcam.com); opening a popup must lazy-load
   hls.js and play LIVE VIDEO (`video.videoWidth > 0` and `currentTime` advancing) from
   `https://{u}/{f}/public/hls/{s}.m3u8`; closing the popup must destroy the stream
   (`activeHls === null`) so streams never pile up. Camera locations + stream codes are a
   baked constant (the GetCameras endpoint only allows the city's own origin — do not "fix"
   by fetching it in-browser); only the streams are live.
9. **Regional weather map = Windy.com only (ECCC WMS map removed 2026-07-13 at user
   request)** — there is NO second Leaflet map, no `#wxmap`, no `#wxLayerPanel`, no radar
   control bar, no GeoMet WMS layers, no lightning note, and no `#wxMapTypeSel` switch; do
   not reintroduce them. The "Regional weather map — forecast animation (Windy.com)" panel
   sits under the drainage map (left column) with `#wxWindyBlock` ALWAYS visible and the
   `#windyFrame` iframe `src` set directly in the HTML (no lazy `data-src`), default
   `overlay=thunder` so the embed's own timeline plays the multi-day FORECAST animation
   out of the box. `#windyBtns` order puts forecast-animating overlays first (Rain &
   thunder default-active, then Wind / Rain accum. / Temperature / Clouds / Satellite),
   with "Radar (live only)" last — Windy's radar overlay is short-range observed history
   (~1 h platform limit), which is why it is not the default; don't let a future edit
   collapse that distinction. Forecast playback is VERIFIED working: pressing the embed's own
   play button on the thunder overlay advanced the timeline ~3 days (Wed 10 PM → Sat 8 AM) in
   one loop. The embed has NO autoplay — `play=1`/`autoplay=1` URL params were tested live and
   ignored, and the iframe is cross-origin so its play button cannot be clicked from the page;
   that's why the panel leads with a prominent accent callout telling the user to press the
   bottom-left play button (do not remove it, and do not burn time re-hunting for an autoplay
   param). "Forecast radar" does not exist in the embed — radar is observed imagery only;
   Rain & thunder is the predicted-precipitation view. `lang=en` stays pinned in the src. The drainage map's layer
   toggle is a constant checkbox panel (`#layerPanel` — defaults ON: district fill, rain
   gauges, 311 flood reports).
9b. **First-map segmented buttons (replaced dropdowns 2026-07-13)** — "Map shows"
   (`#mapMetricBtns`: Storm severity / 311 report density) and "Boundary"
   (`#boundaryBtns`: Districts (15) / Neighbourhoods (407)) are `.segbtn` button groups,
   applied immediately on click with the `.active` class following the selection —
   clicking density must reveal `#densityControls` + the density note and load the
   boundary polygons; there are no `#mapMetricSel`/`#densityBoundarySel` selects anymore.
   The Tracking (rain/snow) header select remains a dropdown and must still swap the 311
   layer label without errors (the ECCC radar-product swap inside `setTrackMode` is gone).
8c. **Next-24h forecast column + PRE-INSPECT (2026-07-14)** — the table's 5th column is the
   per-station next-24h precipitation forecast (`stationFcCell`, amber mm bar), fed by ONE
   batched Open-Meteo multi-point call (`loadStationFc`, all 23 coords comma-separated,
   30-min cache), sortable via `data-k="fc"` (numeric, desc-first). It REPLACED the in-table
   session sparkline (removed; the session line chart still lives in each gauge's dot
   popup — don't re-add a sparkline column). "Storm ponds (SWM facilities)" is a default-off
   layer (`72ee-mmkx`, 370 teal centroid markers; static inventory — NO real-time pond/sewer
   capacity feed exists on the open portal, that's EPCOR-internal SCADA; the popup says so).
   When `loadForecast` sets `fcExtreme`, the priorities list gains a PRE-INSPECT item
   (check pumps/ponds/catch basins for blockage BEFORE the storm); it disappears when the
   forecast calms. District popups' crew context includes the pond count.
9c. **Area search (`#areaSearch`, added 2026-07-14)** — type-ahead over 15 districts +
   407 neighbourhoods (nei polygons lazy-load on first focus via `loadNeighbourhoods`).
   ≥2 chars shows up to 12 matches in `#areaSuggest`; click or Enter calls `gotoArea()`,
   which MUST use `map.fitBounds(b, {..., animate:false})` — animated fitBounds was found
   to silently revert the zoom in this environment (a stuck CSS transition), so do not
   remove `animate:false` as a "cleanup." If the target's polygon layer is currently
   displayed (district fill in severity/density-district mode, or neighbourhood fill in
   density-neighbourhood mode) its real popup opens; otherwise a temporary dashed blue
   outline (`flashOutline`, auto-removed after 8s) + name popup appears — verify both
   paths (e.g. search a neighbourhood while in severity mode → dashed outline, not a
   silent no-op). Arrow keys move the `.hi` highlight, Escape closes, click-outside closes.
10. **Layout** — no horizontal page scroll. Page order: header → KPI tiles → citywide alert
   bar → Shift brief → Priorities → weather forecast panel → tabs. KPI tiles are a two-part
   `.tiles2` flex: LEFT `.tbig` column = "Weather alerts — Edmonton" + "Next 24 h — citywide
   forecast" rendered LARGE (38px values); RIGHT `.tsmall` grid = the other 5 tiles (worst
   gauge · gauges needing attention · 311 flood reports · pumps to check · road closures).
   No band-card strip (band framework lives in the alert bar, priorities, table chips,
   footer). Below the forecast panel is a single always-visible view (no tabs):
   LEFT column = drainage map panel stacked over the Windy regional weather panel,
   RIGHT = stations table, side-by-side flex that wraps on narrow screens.
   The drainage map sizes correctly at creation since nothing is ever tab-hidden;
   check `map.getSize()` and that gauges/districts/popups all work. `lang=en`
   stays pinned in the Windy iframe src so labels are English. Header: the data-as-of
   stamp + "Refresh now" button sit UNDER the title (left-aligned, `.stamp` in full ink, not
   gray). The stations table is `table-layout:fixed` with baked default widths and has 7
   columns: Station (each code is an `.stlink` — clicking opens the `#camModal` overlay with
   the 4 NEAREST live traffic cameras, nearest-first with ~km, each stream its own Hls
   instance in `modalHls[]`, ALL destroyed on close) · District · "311 · closures" (stacked
   district counts from `districtCtx()`: F n × 311 over ! n closures + EMERG bold) · "24h
   rain (mm)" (compact mm bar, max ~36px) · "24h line" (per-row `sparkline()` SVG from the
   same session buffer as the popup chart, current total labelled at the right end point;
   "builds over session" under 2 samples; not sortable) · "Rating · action now" (ONE stacked
   column: band chip on top, action text + session trend arrow below) · Feed says. Every
   header has a `.colrz` drag handle — dragging resizes that column and must NOT trigger the
   sort. The map `#layerPanel` ends with a "Clear all layers" button that UNCHECKS every
   layer toggle (map legend only — it must not touch the table filter bar).

11. **Tracking mode (rain ⇄ snow)** — a "Tracking" select in the header (next to Refresh)
   switches the seasonal objective; default = rain May–Oct, snow Nov–Apr, persisted in
   localStorage `edm_track_mode`. The `TRACK` config drives everything: snow mode swaps the
   ECCC radar to `RADAR_1KM_RSNO` (+`RADAR_COVERAGE_RSNO`, frames re-init, stack cleared),
   re-queries 311 with snow/ice categories (`%SNOW%`, `ICY%` prefix — NOT `%ICY%`, which
   matches "Policy" — `ICE %`, `%WINDROW%`; red = icy conditions instead of underpass
   flooding; KPI title, layer label, popups all relabel), and the Open-Meteo brief/crew
   window computes on `snowfall` in cm (extreme at ≥10 cm total / ≥3 cm/h) with
   snow-worded phrases. Snow mode ALSO (2026-07-16): retitles the first map via `cfg.mapTitle`
   ("Winter conditions — 311 snow/ice reports (rain gauges idle off-season)" — it must never
   claim snowfall gauge measurements, the network is rain-only), relabels the map legend's 311
   phrase + red meaning (`#legend311Phrase`/`#legend311Red`, via `applyTrackLabels`), and swaps
   the Windy embed to the Snow accum. forecast overlay (`cfg.windyOverlay` → `setWindyOverlay`,
   a "Snow accum."/`snowAccu` button now sits second in `#windyBtns`; persisted snow mode also
   applies it at startup). Switching back to rain restores all of it in place (verified live
   both directions). The gauge table is rain-only either way — off-season the seasonbar
   explains the quiet feed.

12. **Weather 311 density mode (first map, "Map shows" buttons)** — a SEPARATE fill mode from
   live storm severity, for "where do people report weather issues most" investigation,
   independent of the rain/snow TRACK mode. Density and the historical heatmap BOTH filter to
   `WEATHER311_WHERE` (2026-07-16, user request — all-category counts dwarfed the map markers
   and read as inconsistent): flood/ponding/drainage + snow/windrow/plow/blading + icy/ice
   conditions + extreme weather, BOTH seasons, category list built from the live distinct
   `service_description` values (word-anchored patterns — plain `%ICY%`/`%ICE%` match
   "Policy"/"License", the documented trap). The heatmap's total and the density map's
   neighbourhood sum must agree exactly for the same range (verified: 45,156 = 45,156); the
   14-day marker layer stays mode-specific (flood OR snow/ice) per §7g and is narrower still —
   that difference is documented in `#wardNoteDensity`, don't "fix" it. `#mapMetricSel` toggles
   `mapMode` severity ⇄ density; density reveals `#densityControls` (period + boundary selects +
   gradient legend) and swaps the note (`#wardNoteSeverity` ⇄ `#wardNoteDensity`).
   - **Period** — a `#densityPresetSel` dropdown (Past week / month / 3 months / year (default) /
     2 years / All available since Jan 2023 / Custom range) for one-click selection, PLUS the
     free `#densityFrom`/`#densityTo` `<input type=date>` pair for anything else — both exist
     together, neither replaced the other. Picking a preset calls `applyDensityPreset(key)`,
     which computes From/To from `todayLocalDate()` and writes both inputs; editing From or To
     by hand snaps the preset select to "Custom" (so it never shows a stale preset label next to
     a manually-edited range). Both date inputs' `min` is `DATA_311_START_DATE` (2023-01-01) and
     `max` is `todayLocalDate()` — the 311 feed's live `min(date_created)` is confirmed 2023-01-01
     via SoQL, so a 10-year window does NOT exist in the source and the pickers physically can't
     go earlier or past today (the "All available" preset maps to this floor, not to 10 years).
     `densityCutoff()`/`densityUntil()` build the SoQL bounds from `densityFrom`/`densityTo`
     (`until` = start of the day AFTER "to", so "to" is inclusive). A `densityReqId` generation
     counter drops any fetch response that isn't from the most-recently-issued request — editing
     From then To (or picking a preset right after another) fires overlapping fetches, and
     without this guard a slower stale wide-range response can land after and silently overwrite
     the correct narrower one (caught live: summed to ~8x the correct count for a Jan–Jun 2023
     test range until the guard was added — verify a rapid two-field edit still lands on the
     right numbers).
   - **Standardized fill (rate, not raw count)** — `densityRate(v) = v / densityDays() * 30`
     converts every count to "reports per 30 days" BEFORE coloring or computing `densityMax`, in
     both `districtStyle`/`neiStyle` and the legend. This is intentional: a raw-count fill makes
     a 2-year range look uniformly "worse" than a 1-week range purely because it accumulated
     longer, which isn't a real density signal — dividing by the actual span the user picked
     (`densityDays()`, inclusive of both endpoints) keeps colors comparable across any Quick
     range/custom choice. The legend (`#densityLegend`) and district/neighbourhood popups show
     BOTH numbers — the raw count (what actually happened) and the standardized `/30d` rate (why
     it's that color) — never only the rate, since the raw count is what a supervisor would
     actually want to know actually happened. `#densityTotal` (see below) stays a raw sum on
     purpose — it's a total, not a comparable-across-ranges rate, so don't "standardize" it too.
   - **Boundary** (`#boundaryBtns` segmented buttons, one click applies): Districts (15,
     `avbh-ga6n`'s `district_name` — same
     polygons as severity mode, restyled) or Neighbourhoods (407, `65fr-66s6`, lazy-loaded once
     via `loadNeighbourhoods()`). District-level counts are a ROLLUP of neighbourhood counts
     through each neighbourhood polygon's own `district` field (verified to match
     `avbh-ga6n.district_name` exactly, incl. "Southwest"/"Rabbit Hill" which have no gauge and
     so never appear in the severity-mode station filter) — 311 records carry a `neighbourhood`
     name but no district field, so this join is required, not optional.
   - **Density popups are concise** (user request 2026-07-16): count first ("<b>488</b> weather
     311 reports"), NO date range (it already sits in `#densityTotal` above the map), and the
     rate line reads "≈ 40.0 per 30 days — sets the fill color" (never the cryptic "40.0/30d");
     the legend max also says "per 30 days". Popups on `#map` are semi-transparent
     (`color-mix(… 78%, transparent)` + blur on `.leaflet-popup-content-wrapper`/`-tip`) so the
     clicked, highlighted polygon stays visible beneath them.
   - **Fill**: continuous gradient (`densityColor`, 4-stop yellow→orange→red→magenta interpolation
     over `v/densityMax`), NOT the fixed severity bands — `densityMax` is recomputed per
     boundary+period so the ramp always spans the current data. Legend (`#densityLegend`) shows
     the actual 0–max range as text + a CSS gradient bar.
   - **Headline total** (`#densityTotal`, "N reports · <range label>") — the user asked for the
     311 numbers to visibly sync with the picker, so this is computed fresh in
     `updateDensityLegend()` every time from whichever of `densityByNei`/`densityByDistrict`
     matches the CURRENT `densityBoundary` (never a cached/independent count) and paired with
     `densityRangeLabel()`. It must move immediately whenever From, To, or Boundary changes —
     that live movement is the check. District vs neighbourhood totals can differ slightly
     (a neighbourhood with no `district` mapping counts in the neighbourhood total but not the
     district rollup) — that's expected, not a bug, since each total matches its own map fill.
     This is entirely separate from the "311 flood reports (14 d)" KPI tile / table column,
     which stay a fixed 14-day drainage-relevant signal by design (see §7g) — do not merge the
     two or make the KPI tile follow this picker.
   - **Layer swap**: `updateBoundaryLayer()` keeps exactly one of `districtGeoJSON` /
     `neiGeoJSON` inside `wardLayer` at a time; switching back to severity mode must restore
     `districtGeoJSON` + call `restyleWards()` (severity colors), and switching mode/boundary
     must never leave both layers on the map simultaneously.
   - **Data caveat (verified against a live cross-check, do not remove)**: for a 1-year window,
     total 311 volume was ~940k rows but the neighbourhood-keyed rollup summed to only ~425k —
     NOT a bug: ~516k rows in that window have a null `neighbourhood` field (city-wide/unaddressed
     request types) and are correctly excluded, since they can't be placed on a neighbourhood
     map. `#wardNoteDensity` must keep stating this ("fill shows only the geolocatable subset,
     not the full request volume") — cutting it would overclaim coverage.
   - Cross-check: `q7ua-agfg.json?$select=neighbourhood,count(*) as n&$where=date_created>='<cutoff>'&$group=neighbourhood&$limit=1000`
     summed over non-null `neighbourhood` rows must match `densityByNei`'s value sum for that
     period; spot-check 2–3 named neighbourhoods against the dashboard's popup count.

13. **Historical heatmap (weather-related 311 reports per month)** — full-width panel (titled
   "Historical patterns — weather-related 311 reports per month", filtered by
   `WEATHER311_WHERE` — see §12) between the filter bar and
   the maps/table row, for historical pattern review. CSS grid (`#hmGrid`): y = district or
   neighbourhood rows (sorted by total desc), x = one column per month of the selected range,
   cell fill = that month's 311 count via `densityColor` against the grid's own max (raw counts,
   not the /30d rate — months are already equal spans, so no standardization needed); hover
   title = exact count.
   - **Fixed cell size**: columns are a constant 34px (`repeat(n, 34px)`, NOT `1fr`) so a
     2-month range renders the same cell footprint as a 43-month one and the grid scrolls
     horizontally when months overflow — the user explicitly rejected range-dependent cell
     stretching. Edge columns can be partial months (range starts/ends mid-month) and read
     lighter; the panel note says so.
   - **Shared controls, one source of truth, TWO-WAY sync**: it follows the SAME Quick range /
     From–To / Boundary controls as the density map (which is why `#densityControls` is ALWAYS
     visible, in both map modes — do not re-hide it in severity mode) plus the filter bar's
     District select (`fDistrict` zooms rows; `fClear` restores). Sync is bidirectional:
     (a) touching Quick range / From / To / Boundary flips the map into density mode
     (`syncDensityMap`) so those filters visibly apply to map + heatmap together — the
     live-severity map/table cannot be date-filtered (snapshot feed), which is the only reason
     they don't respond to the range; (b) clicking a heatmap ROW LABEL applies that row's
     district to `fDistrict` (table rows + map markers + heatmap zoom all at once; clicking the
     active district again clears it — labels carry `data-d`, delegated click on `#hmGrid`).
     Fetch (`loadHeatmap`, own `hmReqId` stale-response guard) happens only on range changes —
     boundary/district changes re-render client-side from the same `hmData`.
   - **Polygon click = selection** (2026-07-16): clicking a DISTRICT polygon runs the same
     `toggleDistrictSelection()` path as the heatmap row labels (filter + table + markers +
     vivid/faded styling; click-again clears; popup still opens). Clicking a NEIGHBOURHOOD
     polygon spotlights ONLY that neighbourhood (`toggleNeiSelection`/`selectedNei` — user
     explicitly rejected parent-district selection here): it goes vivid with the blue border,
     every other neighbourhood fades to its own muted color, and it does NOT touch `fDistrict`
     (visual spotlight only — finer than the table/heatmap's district granularity). The two
     levels never stack: `selectedNei` takes styling precedence in `neiStyle`, and any
     fDistrict change or fClear resets `selectedNei = null`.
   - **Collapsible help notes**: each map section's instructional text sits in a
     `<details class="helpnote">` collapsed by default ("ℹ How to read this map…", "ℹ How to
     read this heatmap", "▶ To play the multi-day forecast…") — data leads, instructions fold;
     the ward severity/density inner notes still swap by mode INSIDE the one details wrapper.
   - **Selection highlight (map polygons + row labels)**: while a district is selected
     (`selectedDistrict()`, from `fDistrict` with the "near " prefix stripped), the chosen
     district keeps its full fill color plus a blue accent border (`#2a78d6`, weight 2.5) and
     every other polygon fades via `muted(style)` — it KEEPS ITS OWN fill color at ~35% of its
     normal fillOpacity (floor .12) with a light `#b9b8b0` border, NOT a flat gray wash (the
     user explicitly asked for "more muted but not fully muted": the citywide pattern must stay
     faintly readable behind the selection; verified live — muted Scona keeps its rating-0
     yellow `#fee79a` at op .19). Applied via `withSelection()` in `districtStyle` (both
     severity and density branches) and the district-aware `neiStyle(name, district)`
     (in-district neighbourhoods keep color with blue borders, out-of-district ones fade the
     same way). `restyleBoundaries()` re-applies both layers on fDistrict change and fClear;
     the matching heatmap row labels get `.hm-lab.active` (blue, bold). Clearing the selection
     must restore normal styling everywhere (verified: Scona back to `#898781`/weight 1/op .55).
   - **Testing trap (Browser pane)**: Chrome restores prior form values (preset select, date
     inputs, fDistrict) asynchronously after navigation and FIRES change events — a scripted
     test can be silently overridden seconds later by the restored user state. Run assertion
     sequences in ONE page session without reloading between steps, or expect phantom state.
   - **Query**: `q7ua-agfg` grouped by `neighbourhood, date_trunc_ym(date_created)`,
     `$limit=50000` (full 2023→now range is ~18k grouped rows). First cold-cache call on a new
     range can take 5–10 s ("loading heatmap…" persists) — wait before declaring it broken; the
     warm repeat is ~300 ms.
   - **Row cap**: neighbourhood boundary with no district filter shows top 30 by total with an
     explicit "N more — pick a district to zoom" note (407 rows are unreadable); district
     boundary always shows all 15. District rollup uses the same nei→district join as the map.
   - **No severity-by-month**: the gauge feed is a snapshot with NO published history, so a
     severity heatmap cannot be built from the source — the panel note says so explicitly
     ("this dashboard never invents history"). Do not fake one from the session buffer.
   - Cross-check: pick one cell (e.g. DOWNTOWN · a full past month) and curl
     `$select=count(*)&$where=neighbourhood='X' AND date_created>='<m1>' AND date_created<'<m2>'
     AND <WEATHER311_WHERE>` — must equal the cell tooltip exactly (verified live with the
     weather filter: DOWNTOWN Nov 2025 = 7 both ways; omitting the filter gives all-category
     counts, ~10x larger, and is the wrong comparison).

## Invariants to respect when editing the dashboard

- Numbers are parsed from strings before sort/math — check any new metric does the same.
- Band colors are status colors and never carry meaning alone (rating number + band label
  always accompany them).
- A missing/stale gauge renders as a flagged blind spot, never as 0 mm.
- Trend UI must state it is session-local ("this browser only — not official history").
