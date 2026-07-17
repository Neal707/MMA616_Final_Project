---
name: edmonton-rainfall-metadata
description: Metadata library for the Edmonton 24 Hour Rainfall Totals and Storm Severity Ratings dataset (x4rm-mppc). Use this BEFORE writing any API query, metric, chart, or code that touches the rainfall feed — it documents the exact endpoint, every field with its type quirks, the storm rating scale, and the traps that produce silently wrong numbers (string numerics, the $select 400 error, the midnight-reset timestamp). Also use when the user asks what a field means, why a number looks wrong, or how fresh the data is.
---

# Edmonton Rainfall Dataset — Metadata Library

Dataset `x4rm-mppc`, City of Edmonton Open Data. **One row = one rain-gauge station's current
snapshot** (~23 stations, RG17–RG62). The feed overwrites in place every ~15 minutes and keeps
**no history** — trends require client-side accumulation (the dashboard's localStorage buffer).

## Endpoint — use exactly this

```
https://data.edmonton.ca/resource/x4rm-mppc.json?$$exclude_system_fields=false&$limit=100
```

- `$$exclude_system_fields=false` is the only way to get `:updated_at` (the freshness stamp).
  `$select=:updated_at,*` returns **HTTP 400** — verified live 2026-07-12.
- The v3 URL (`/api/v3/views/x4rm-mppc/query.json`) returns the same data in a noisier envelope; avoid.
- Full pull is ~23 rows — never paginate; prefer one pull + client-side math over server aggregates.

## Fields

| Field | Type as delivered | Meaning | Trap |
|---|---|---|---|
| `station` | string | Gauge ID (`RG17`…) | The join/display key |
| `_24h_total` | **string** | Rolling 24h rainfall, mm | Parse to float — string sort puts `"5"` above `"40"` |
| `storm_rating` | **string** | Severity 0–8 | Parse to int |
| `storm_rating_description` | string | Plain-language consequence | Render verbatim; it is the city's own wording |
| `amount_legend` | string | Map icon band (`0 - 5 mm` … `40 mm or more`) | Cosmetic only — compute bands from `_24h_total` |
| `date_time` | datetime | When reported conditions were recorded | **Not freshness** — resets to midnight daily |
| `date` / `time` | datetime / string | Components of the above | Same caution |
| `location` / `geometry_point` | object | WGS84 lat/long (location values are strings) | Parse floats for map math |
| `row_id` | string | `{station}_{timestamp}-06:00` | Confirms Mountain time |
| `:updated_at` | datetime, **UTC** | Socrata row update stamp | **The real freshness signal**; convert to America/Edmonton for display. Stamps are batch-level (all rows match) |
| `:@computed_region_*` | number | Boundary joins, unlabeled IDs | Not used |

## Storm rating scale → action bands

| Rating | Feed description (verbatim) | Band → action |
|---|---|---|
| 0–2 | damp roads … gutters/swales flowing | Normal — routine monitoring |
| 3 | "Ponding near catchbasins, sewers filling" | Monitor — watch, pre-position if rising |
| 4 | "Extensive ponding … sewers may back up" | Dispatch — crews to catch basins, pump checks |
| 5 | "Streets may flood, underpasses … may start to flood" | Barricade — close underpasses & low areas |
| ≥6 | "Flooding widespread" | Escalate — notify Emergency Management |

Ratings 1, 7, 8 have not been observed live; handle any integer gracefully (band by threshold,
show the feed's own description string).

## Rules that prevent wrong outputs

1. **Silence ≠ zero.** Gauges transmit only after 2 mm accumulates; the station set can vary.
   A missing/stale station during a storm is a blind spot — flag it, never impute 0.
2. **Seasonal.** Measured May–October only. Off-season quiet = feed inactive, not dry.
3. **Gauge-location-only.** The city's own caveat: nearby areas may differ. Don't interpolate
   between gauges or claim neighbourhood certainty.
4. **Rolling 24h window is a floor.** Storms longer than 24h can have had higher totals/ratings.
5. **Trend claims must state buffer depth** ("based on N snapshots over X min") and be labeled
   session-local, never official history.
6. **No history API exists — don't search for one.** `7fus-qa4r` ("Rainfall Gauge Results") is
   daily-grain, ends 2019-10, and uses a different gauge network (P-series IDs, no join to
   RG-series). Verified live 2026-07-12. Intra-day history comes only from the session buffer.

**Companion datasets** (full evaluation table in the library file): `yhez-gf32` pump stations
(STORM/COMBINED only), `k4tx-5k8p` live traffic disruptions, and `q7ua-agfg` 311 requests
(flood-related descriptions only, last 7 days — updates DAILY, locations are neighbourhood
centroids, so context only, never the alert signal; includes an "Underpass Flooding Dispatch"
description) are merged as map context layers. Evaluated and **rejected** — don't re-propose
without rechecking: hourly ECCC weather (`ib2b-3mi4`, no precip field + corrupt dates); water
levels (`cnsu-iagr`, stale — max reading 2026-06-03, rechecked 2026-07-12); the 2014 Flood
Mitigation ponding-depth map layers (`fkxj-864d` etc. — SODA exposes zero columns and the
GeoJSON export returns an empty FeatureCollection, verified 2026-07-12; no machine-readable
underpass/low-point inventory exists on the portal).

Deeper detail (full cautions, SoQL patterns, provenance): `knowledge/edmonton-rainfall-library.md`.
Personas and metric definitions: `CLAUDE.md`.
