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
4. **Trend buffer** — inject a synthetic snapshot into localStorage
   (`edm_rain_buffer_v1`, shape `[{t:<ms>, st:{RG17:[mm,rating],…}}]`) dated ~50 min back with
   offset totals, call `load()`, confirm rising/falling arrows, sparklines, and the buffer-depth
   note appear. **Remove the synthetic snapshot afterwards** — it poisons real trends.
5. **Failure path** — stub `window.fetch` to reject, call `load()`, confirm the failbar shows
   "DATA UNAVAILABLE … do not assume no rain" and the stamp reads "feed unreachable". Restore
   fetch and reload.
6. **Popup chart** — with a multi-point buffer injected (step 4), `markerLayer.eachLayer` →
   `openPopup()` on a marker must show an SVG line with one path point per buffered reading and
   the "session data … not official history" caption; with a single-snapshot buffer it must show
   the "builds while the dashboard stays open" message instead of an empty plot.
7. **Diverging marker scale** — marker fill by rating: 0 `#2a78d6`, 1 `#6da7ec`, 2 `#b7d3f6`,
   3 `#9a9992` (gray pivot), 4 `#ec835a`, 5 `#d03b3b`, ≥6 `#8a1616`; rating number stays on the
   marker (dark ink on the light fills).
8. **Layout** — no horizontal page scroll; map has height; 4 KPI tiles; 5 band cards.

## Invariants to respect when editing the dashboard

- Numbers are parsed from strings before sort/math — check any new metric does the same.
- Band colors are status colors and never carry meaning alone (rating number + band label
  always accompany them).
- A missing/stale gauge renders as a flagged blind spot, never as 0 mm.
- Trend UI must state it is session-local ("this browser only — not official history").
