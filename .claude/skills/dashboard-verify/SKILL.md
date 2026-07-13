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
   (normally 23). Band-strip counts must sum to the same number.
2. **KPI cross-check** — recompute independently with curl and compare:
   ```
   .../x4rm-mppc.json?$select=max(_24h_total)                     → Peak KPI (compare numerically — API max is a string max, so recompute client-side if totals span digit lengths)
   .../x4rm-mppc.json?$select=storm_rating,count(*)&$group=storm_rating → band counts, stations ≥4
   ```
3. **Freshness stamp** — page's "Data as of" equals newest `:updated_at`
   (`$$exclude_system_fields=false`) converted to America/Edmonton.
4. **Trend buffer** — key `edm_rain_buffer_v2`, shape `[{t:<ms>, st:{RG17:[mm,rating],…}}]`.
   `t` is the FEED PUBLISH time (batch `:updated_at`), and `pushBuf` only appends when the
   stamp advances — page refreshes between publishes must NOT add points (that regression
   showed every station as "steady +0.0 mm/5m"). Inject a synthetic point dated ~40 min
   before the current one with offset totals, `memBuf=null; load()`, confirm rising/falling
   arrows, sparklines, and a "N feed publishes over X min" note. **Remove the synthetic
   snapshot afterwards** — it poisons real trends.
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
7b. **Priorities panel** — ordered duty list derived from state: ESCALATE (≥6) → BARRICADE (5)
   → DISPATCH (4) → PRE-POSITION (rising 3s) → REASSESS (falling ≥4) → VERIFY (blind spots)
   → ROUTE AROUND (emergency disruptions count). Grouped by district. Empty state reads
   "ROUTINE MONITORING". Normal band card shows count only, no station list.
7c. **Sortable columns** — clicking any header sorts (numeric columns start descending);
   arrow indicator on the active header; blind rows stay pinned on top regardless of sort.
   There is NO "Updated" column — freshness lives in the header stamp only.
7d. **Forecast panel** — Open-Meteo next-24h: headline (total mm, peak hour, max gusts) +
   24 precip bars with °C labels; on API failure it says "decide from live gauges and radar",
   never blocks the rest of the dashboard.
8. **Filters** — Station / Rating / Feed-says selects filter the table rows AND gauge dots
   (selecting rating 5 must shrink both); ward fills and citywide KPIs must NOT change;
   Clear restores all rows; selections survive an auto-refresh.
9. **Weather overlays (ECCC GeoMet WMS)** — enabling "Weather radar (animated)" must show the
   radar control bar, populate ~31 frames from GetCapabilities (rolling ~3 h at 6-min steps —
   the service offers no 24 h loop; do not "fix" that), and play must advance the MT time label
   with `TIME=` in the WMS tile URLs. Alerts + lightning are latest-only overlays. The Windy
   iframe must have no `src` until its `<details>` panel is opened.
10. **Layout** — no horizontal page scroll; map has height; 4 KPI tiles; 5 band cards.

## Invariants to respect when editing the dashboard

- Numbers are parsed from strings before sort/math — check any new metric does the same.
- Band colors are status colors and never carry meaning alone (rating number + band label
  always accompany them).
- A missing/stale gauge renders as a flagged blind spot, never as 0 mm.
- Trend UI must state it is session-local ("this browser only — not official history").
