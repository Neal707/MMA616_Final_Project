---
name: calgary-311-data-dictionary
description: Field-level reference for Calgary 311 iahh-g8bj — grain, key fields, types, known values
metadata:
  type: reference
---

# Calgary 311 Data Dictionary

Dataset: `iahh-g8bj` — Calgary 311 Service Requests, full history.
API: `https://data.calgary.ca/resource/iahh-g8bj.json` (Socrata SoQL).
Grain: **one row = one service request**.
Encoding: dates ISO-8601 with time component (`...T00:00:00.000`); coordinates EPSG:4326.

---

## Key Fields

| Field | Type | Description | Notes |
|---|---|---|---|
| `service_request_id` | string | Unique request identifier | Primary key |
| `requested_date` | datetime | When the request was submitted | Use for inflow counts and backlog aging |
| `updated_date` | datetime | Last update to the record | Not reliable for currency — use `MAX(requested_date)` |
| `closed_date` | datetime | When the request was closed | NULL for open requests; required for resolution metrics |
| `status_description` | string | Current status | Values: `Open`, `Closed`, `Duplicate (Open)`, `TO BE DELETED` |
| `service_name` | string | Service/category type | e.g., "Graffiti Removal", "Pothole Repair" |
| `agency_responsible` | string | City department handling it | Can be used for agency-level triage |
| `comm_name` | string | Community name | Use for community breakdowns; may be null |
| `comm_code` | string | Community code | Stable identifier; rank by code, label with name |
| `longitude` | float | WGS84 longitude | May be null |
| `latitude` | float | WGS84 latitude | May be null |
| `point` | object | Socrata geo point | Combines lat/long |
| `address` | string | Street address | **100% NULL — do not use** |
| `source` | string | Intake channel | Not previously documented; discovered 2026-07-02. Untested downstream — verify distinct values before building the intake-channel panel on it |
| `:@computed_region_4a3i_ccfj` | number (1–4) | Socrata-computed point-in-polygon join against Calgary's official City Quadrants boundaries | No text label in-resource — id→NW/NE/SW/SE mapping is **inferred** (centroid-vs-downtown comparison), live-spot-checked against 5 known communities: `1`=SW (Aspen Woods), `2`=NW (Tuscany), `3`=SE (Forest Lawn, Auburn Bay), `4`=NE (Rundle). Used to group the dashboard's Community filter dropdown. See `data-cautions.md` |
| `:@computed_region_kxmf_bzkv` | number | Computed region join against a separate "Calgary Communities" boundary layer (296 distinct values vs. `comm_name`'s 325) | Not used — flagged as a possible cleaner community key if `comm_name`'s ~52 code-only placeholders (e.g. `01B`) ever need resolving |
| `:@computed_region_4b54_tmc4` / `:@computed_region_p8tp_5dkv` | number | Ward Boundaries (current / 2013–2017) | Not used by the dashboard |

---

## Status Values

| Value | Meaning | Include in backlog? |
|---|---|---|
| `Open` | Active, unresolved | Yes (with `closed_date IS NULL`) |
| `Closed` | Resolved | No (exclude from open backlog) |
| `Duplicate (Closed)` | Flagged as duplicate and closed | No (resolved, exclude) |
| `Duplicate (Open)` | Flagged as duplicate, not yet closed | No (exclude — not actionable) |
| `TO BE DELETED` | Data garbage | No (exclude always) |

**Correct open backlog filter:**
```sql
WHERE closed_date IS NULL AND status_description = 'Open'
```

---

## Time Dimensions

| Purpose | Field(s) |
|---|---|
| Inflow volume / aging of open requests | `requested_date` |
| Resolution time of closed requests | `closed_date - requested_date` |
| Currency check | `MAX(requested_date)` |

---

## SoQL Query Patterns

**Current open backlog count:**
```
SELECT COUNT(*) WHERE closed_date IS NULL AND status_description = 'Open'
```

**Weekly inflow by service × community (last N weeks):**
```
SELECT service_name, comm_name, date_trunc_ym(requested_date) AS week,
       COUNT(*) AS inflow
WHERE requested_date >= '[start]'
GROUP BY service_name, comm_name, week
```

**Currency stamp:**
```
SELECT MAX(requested_date)
```

**Pagination:** Socrata default limit is 1,000 rows. Use `$limit` and `$offset` for full pulls. `scripts/pull.py` handles this.
