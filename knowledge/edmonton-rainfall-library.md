---
name: edmonton-rainfall-library
description: Field-level reference and data cautions for Edmonton x4rm-mppc — 24 Hour Rainfall Totals and Storm Severity Ratings (live snapshot dataset)
metadata:
  type: reference
---

# Edmonton Rainfall & Storm Severity — Dataset Library

Dataset: `x4rm-mppc` — **24 Hour Rainfall Totals and Storm Severity Ratings**, City of Edmonton Open Data.
API (use this one): `https://data.edmonton.ca/resource/x4rm-mppc.json` (Socrata SODA, CORS-enabled, SoQL).
Alternate (avoid for the dashboard): `https://data.edmonton.ca/api/v3/views/x4rm-mppc/query.json` — same data, noisier envelope.

**Grain: one row = one rain-gauge station's *current* snapshot.** ~23 stations (RG17–RG62), one row each.
This is a **live snapshot, not a history** — the feed overwrites in place roughly every 15 minutes and keeps no past records. Any trend view must be accumulated client-side.

Official description highlights (dataset metadata, verified 2026-07-12):
- "Website is updated every 15 minutes."
- "Data is sent once 2 mm is accumulated in each gauge, so timing will vary."
- "Rainfall is measured between May and October; snowfall is not measured."
- "Storm ratings and rainfall are based on data at the gauge location only — nearby areas may experience differing conditions."

---

## Fields

| Field | Type | Description | Notes |
|---|---|---|---|
| `row_id` | string | Composite key: `{station}_{timestamp}{offset}` e.g. `RG17_20260711152500-06:00` | Confirms Mountain time (−06:00 in summer) |
| `date` | datetime | Calendar date, midnight-truncated (`...T00:00:00.000`) | Do not use for freshness — always midnight |
| `time` | string | Clock time `HH:MM:SS` | Pairs with `date`; see `date_time` caution below |
| `date_time` | datetime | Timestamp of the reported conditions | **Not a reliable freshness stamp** — observed resetting to midnight daily; appears to mark when the reported 24h max conditions were recorded, not when the feed last refreshed |
| `station` | string | Rain gauge ID, e.g. `RG17` | Stable station identifier; the join/display key |
| `_24h_total` | string→number | Rolling 24-hour rainfall total in **mm** | Arrives as a **string** — parse to float before math |
| `storm_rating` | string→number | Storm severity rating, integer scale (0–8; see below) | Arrives as a **string** — parse to int |
| `storm_rating_description` | string | Plain-language consequence of the rating | Written in sewer-capacity language; render verbatim in UI |
| `amount_legend` | string | Display band for the total | Values: `0 - 5 mm`, `5 - 10 mm`, `10 - 20 mm`, `20 - 30 mm`, `30 - 40 mm`, `40 mm or more`. Display-banding only — derive nothing from it; compute bands from `_24h_total` |
| `location` | object | Socrata location: `latitude`, `longitude` (WGS84), empty `human_address` | Use for map pins |
| `geometry_point` | object | GeoJSON Point `[lon, lat]` | Duplicate of `location`; either works |
| `:@computed_region_*` (7 fields) | number | Socrata point-in-polygon joins vs city boundary layers | Numeric IDs only, no labels in-resource — **not used** by the dashboard |
| `:updated_at` | datetime (system) | Row-level Socrata update stamp (UTC) | **The real freshness signal.** Request via `$$exclude_system_fields=false` (note: `$select=:updated_at,*` returns HTTP 400 — verified live) |

---

## Storm rating scale (observed values + descriptions, live 2026-07-11/12)

The rating is Edmonton's sewer-capacity-anchored severity scale. Values observed live with their exact descriptions:

| Rating | Description (verbatim from feed) | Operational band |
|---|---|---|
| 0 | No rain, or minor rain - Roads damp, lawns may still need watering after | Normal |
| 1 | *(not yet observed live — presumed light-rain step between 0 and 2)* | Normal |
| 2 | Road gutters flowing well, grassed swales visibly flowing | Normal |
| 3 | Ponding near catchbasins, sewers filling | **Monitor** |
| 4 | Extensive ponding in low spots, sewers may back up, eavestroughs may overflow | **Dispatch** |
| 5 | Streets may flood, underpasses and low areas may start to flood | **Barricade** |
| 6 | Flooding widespread - streets, low areas. River depends on large areas west of City. | **Escalate** |
| 7–8 | *(not yet observed live — scale reportedly extends to 8 for extreme/river flood events)* | **Escalate** |

Ratings 1, 7, 8 have not appeared in live data during this project; the dashboard must render any unobserved rating gracefully (show the number + the feed's own description string, band it: ≤2 Normal, 3 Monitor, 4 Dispatch, 5 Barricade, ≥6 Escalate).

---

## Data cautions (deletion test: each line, if removed, would ship a wrong output)

1. **Snapshot only — no history.** One row per station, overwritten in place. Any "rising/falling" trend must come from a client-side session buffer (localStorage on each refresh). Never present buffer-derived trends as official history.
2. **Numerics are strings.** `_24h_total` and `storm_rating` arrive as strings; string comparison sorts `"5"` above `"40"`. Parse before sorting or math.
3. **`date_time` is not freshness.** It resets to midnight daily (observed live 2026-07-12: all 23 rows at `T00:00:00`). Use the Socrata system field `:updated_at` (UTC) for the data-as-of stamp, converting to local Mountain time for display.
4. **Silence ≠ zero.** Gauges transmit only after 2 mm accumulates, and the station set can vary from 23. A missing or stale station during a storm is a *blind spot*, not "no rain." Flag it; never impute 0.
5. **Seasonal feed.** No measurements November–April (snowfall not measured). Off-season, the dashboard should say "off-season / feed inactive," not imply drought.
6. **Gauge-location-only readings.** The city's own caveat: nearby areas may differ. Interpolating between gauges overstates confidence — show station points, not smoothed surfaces.
7. **`amount_legend` is cosmetic.** It's the city map's icon band. Compute all bands/thresholds from `_24h_total` directly.
8. **Rolling 24h window can mask long storms.** Metadata: "Higher storm ratings and rainfall totals may have occurred if a storm lasts more than 24h." The total is a floor, not a storm-event total.
9. **Timezone.** Timestamps in `row_id` carry `-06:00` (MDT). `:updated_at` is UTC. Convert consistently; the supervisor thinks in local time.
10. **No companion history dataset exists — do not go looking.** Catalog-searched 2026-07-12: the only candidate, `7fus-qa4r` ("Rainfall Gauge Results"), is **daily**-grain, ends **2019-10-07**, and uses a different gauge network (`P001`–`P3xx` IDs — not the RG-series in this feed; no join key). `jwz2-cge4` covers 2013–2015 only. Intra-day per-station history is available **only** from the dashboard's session buffer.

---

## Companion datasets merged into the dashboard (evaluated 2026-07-12)

| Dataset | Verdict | Use |
|---|---|---|
| `yhez-gf32` Drainage — Pump Stations | **Merged** (static context layer, default off) | 108 stations; filter `type in('STORM','COMBINED')` → 30 relevant. Rating-4 action is "pump checks" — shows which pumps sit near hot gauges |
| `k4tx-5k8p` Traffic Disruptions | **Merged** (live context layer, default off, refreshed each cycle) | Filter `status in('Current','Emergency','Revised')`, keep `Total Closure` + any `Emergency`, date-window client-side. Closures already in effect change barricade/dispatch routing (~87 in effect + 3 emergencies at eval time) |
| `ib2b-3mi4` Weather Data Hourly (ECCC) | **Rejected** | No precipitation field at all, and corrupt `date_and_time` values (max = year 57971) |
| `cnsu-iagr` Water Levels and Flows | **Rejected** | Last row 2026-06-03 (a month stale at eval) and stations are provincial/upstream, not city drainage |
| `7fus-qa4r` Rainfall Gauge Results | **Rejected** (see caution 10) | Daily grain, ends 2019, different gauge network |
| 2014 Flood Mitigation Study layers | **Rejected** | GeoTIFF/map assets, not SODA-queryable |
| `da6r-6gkw` Wards (boundary layer) | **Merged** (choropleth base) | The exact layer Socrata joins `:@computed_region_da6r_6gkw` against; 12 polygons, `simplify_preserve_topology(the_geom,0.0005)` ≈ 27 KB total |
| ECCC MSC GeoMet WMS (`geo.weather.gc.ca/geomet`) | **Merged** (weather overlays, default off) | `RADAR_1KM_RRAI` (animated via TIME param — service keeps ~3 h at 6-min steps, **24 h is not offered**), `ALERTS` (current polygons), `Lightning_2.5km_Density` (~3 h at 10-min steps). Time extent read live from GetCapabilities |
| Windy.com embed | **Merged** (optional iframe, lazy-loaded) | `embed.windy.com/embed2.html` with radar overlay; loads only when the collapsible panel is opened |

## SoQL patterns

```
# Full snapshot (all stations, incl. :updated_at freshness stamp)
https://data.edmonton.ca/resource/x4rm-mppc.json?$$exclude_system_fields=false&$limit=100

# Spot-check aggregates (for verification)
...?$select=max(_24h_total)                         # NOTE: max of string — verify numerically client-side
...?$select=storm_rating,count(*)&$group=storm_rating
```

The full pull is ~23 rows — no pagination needed. Prefer one full pull + client-side computation over server aggregates (which inherit the string-typing problem).
