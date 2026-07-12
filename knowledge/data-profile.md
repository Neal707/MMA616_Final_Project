# Data profile — Calgary 311 Service Requests (`iahh-g8bj`)

The **detailed description**: what one row is, how much there is, what's covered, and what's missing.
All figures recomputed **live from the full source** (SoQL aggregates) on **2026-06-25** — they
drift daily as the feed updates, so treat them as "as of" snapshots, not constants.

## Shape & coverage
- **Grain:** one row = one 311 service request.
- **Volume:** **7,356,371** rows.
- **Date span:** `requested_date` **2010-01-25 → 2026-06-25** (description says "2012→present"; the
  feed actually reaches back to 2010).
- **Recent inflow:** **57,189** requests in the last 30 days ≈ **~1,900/day**.
- **Cardinality:** 1,158 distinct `service_name`, 325 distinct `comm_name`, 93 distinct `agency_responsible`.

## Status & backlog
| status_description | count | share |
|---|---:|---:|
| Closed | 7,163,025 | 97.4% |
| Duplicate (Closed) | 112,737 | 1.5% |
| Open | 78,215 | 1.1% |
| Duplicate (Open) | 2,291 | 0.03% |
| TO BE DELETED | 103 | 0.001% |

- **`closed_date IS NULL` (raw, pre-status): ~76,258** — includes 2,281 rows the actionable filter removes
  (`Duplicate (Open)` + `TO BE DELETED`). Do **not** use this alone for backlog.
- **Actionable backlog (`closed_date IS NULL AND status_description = 'Open'`): 73,977** — the dashboard
  "Open backlog" KPI. (≠ the 78,215 rows merely *labelled* `Open`; defined by both fields, not status text alone.)
  - of which administrative `311 Contact Us`: **26,488** — counted in the total, **excluded** from crew-triage rankings & spikes.
  - stale / unclosed **> 2 years: 34,758** — shown separately, **excluded** from actionable aging rankings (oldest dates to **2014**).
- **Closed with a date (`closed_date IS NOT NULL`): 7,280,107** — the only rows valid for resolution-time analysis.

### Actionable aging (open + status Open, `311 Contact Us` excluded, by `requested_date`)
| Band | Count |
|---|---:|
| Actionable open > 1 year | 19,712 |

All figures verified live via `eval/recompute.py` (C7–C11), currency 2026-06-25. The aged tail is the headline
problem — and because ~47% of the raw open queue is stale (>2yr) and ~26k is administrative `311 Contact Us`,
the *actionable, in-window* queue the dashboard ranks is far smaller than the raw count. That is why the
backlog filter, the 2-year cap, and the 311 exclusion are non-negotiable.

## Null / missingness rates
| Field | Null % | Implication |
|---|---:|---|
| `address` | **100.00%** | unusable — never reference it |
| `comm_name` / `comm_code` | 5.76% | location unknown; keep as `(Unknown)`, exclude from rankings |
| `longitude` (+ `latitude`/`point`) | 5.77% | no map pin for ~6% of rows |
| `closed_date` | 1.04% | **this null set is the backlog** |
| `updated_date` | 1.06% | minor |
| `service_name`, `source`, `agency_responsible`, `location_type` | 0.00% | always present |

`location_type` is effectively a location-known flag: `Community Centrepoint` (94.2%) vs `None`
(5.8%), and `None` lines up with the null `comm_*`/`longitude` rows.

## Distributions that matter for triage
**Intake channel (`source`):** Phone 4,009,292 (54%) · Other 2,022,277 (27%) · App 752,489 (10%) · Web 572,313 (8%).

**Top service types — all-time volume** (drives crews historically):
WRS - Cart Management (252k) · Finance - Property Tax Account Inquiry (199k) · Finance - TIPP
Agreement Request (197k) · CBS Inspection - Electrical (176k) · WRS - Waste - Residential (167k) ·
Bylaw - Snow and Ice on Sidewalk (160k) · Roads - Snow and Ice Control (155k).

**Top service types — currently OPEN** (raw open snapshot ~2026-06-25; counts drift daily — the pinned
actionable top service is cross-checked in `eval/recompute.py` C12):
311 Contact Us (~26.5k — administrative, **excluded** from crew-triage rankings; see `data-cautions.md`) ·
Roads - Sidewalk/Curb & Gutter Repair (~5.1k) · Roads - Signs Missing/Damaged (~4.0k) · Roads - Backlane
Maintenance (~3.3k) · Roads - NEW Signal Request (~3.0k) · Roads - Roadway Maintenance (~2.8k) · Bylaw -
Long Grass/Weeds (~2.6k). → Excluding `311 Contact Us`, the actionable open backlog is dominated by **Roads/Mobility**.

**Top communities — currently OPEN:**
Downtown Commercial Core (1,759) · Beltline (1,575) · Bridgeland/Riverside (1,053) · Saddle Ridge
(1,035) · Bowness (904) · Crescent Heights (811) · Tuscany (789) · Cranston (784).

**Top agencies — currently OPEN:**
OS - Mobility (26,698) · CFOD - Customer Services & Communications (15,690) · PICS - Customer
Service & Communications (10,975) · CS - Emergency Management & Community Safety (6,830) ·
OS - Water Services (2,232) · TRAN - Roads (1,953).

## How to refresh these numbers
Re-run the aggregates against the live source via `scripts/pull.py`, or query directly via SoQL.
Every figure above is a `count()`/`group` SoQL query — independently recomputable, which is how the
eval harness (`eval/recompute.py`) verifies the dashboard.
