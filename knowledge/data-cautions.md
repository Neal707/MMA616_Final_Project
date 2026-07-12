---
name: calgary-311-data-cautions
description: Deletion-test-filtered cautions for Calgary 311 iahh-g8bj — things the agent cannot guess that would cause a wrong output if omitted
metadata:
  type: reference
---

# Calgary 311 Data Cautions

*Applies the Deletion Test: only lines that, if removed, would cause a wrong number to ship.*

Source: plan/recon/dataset-candidates.md + live API exploration. Dataset: `iahh-g8bj` (Socrata, daily).

---

## Backlog Filter — the critical rule

**WRONG:** `WHERE closed_date IS NULL`
**CORRECT:** `WHERE closed_date IS NULL AND status_description = 'Open'`

Why: `closed_date IS NULL` alone includes:
- `Duplicate (Open)` — 2,288 rows (not actionable backlog)
- `TO BE DELETED` — 103 rows (data garbage)

If this line is removed: the headline backlog count is inflated by ~2,400 rows.

---

## Dataset ID — full history only

- **USE:** `iahh-g8bj` (full history)
- **DO NOT USE:** `arf6-qysm` (Current Year view — rolling/capped)

If wrong ID used: trend analysis and trailing norm computation will be silently wrong.

---

## Address Field — 100% null

`address` is always NULL. Use `comm_name` / `comm_code` / `point` for location. Do not reference `address` in any query or UI label.

---

## Aging Cap — 1-year rule (tightened 2026-07-03, was 2 years)

Exclude `requested_date < today - 1 year` from actionable backlog rankings.

These are almost certainly data-quality failures (never officially closed, migrated records) or simply too
old to act on. Show their count separately as "stale/unclosed records" — do not rank them alongside real
backlog.

If this is omitted: ancient data artefacts will appear at the top of the aging ranking, making the list misleading.

**Why tightened from 2 years:** direct human use of the deployed app (not a critic run) judged the 2-year cap
too lenient — items 300–700+ days old read as useless for an actual weekly surge decision. The 2026-07-02
`verifyAfterDays` display tier ("⚠ verify before dispatching" on items over 1 year old, because they're
frequently stalled/duplicate/already-resolved-but-not-closed) was built to caveat exactly that population —
its own existence was evidence the population didn't belong in the actionable window, not a reason to keep
flagging-and-hoping. The verify-flag mechanism is now fully retired since nothing actionable is old enough to
need it.

---

## Rising-demand signal — as built (matches dashboard `scoreGrowth`)

The dashboard flags **where the backlog is growing**: per `service_name` and per `comm_name`, it compares
**new requests against closed** over the last 4 weeks and flags slices whose queue is rising. *(This replaced
the earlier z-score-vs-trailing-8-week spike, which absorbed multi-week ramps into their own baseline and
missed them — e.g. `Bylaw - Long Grass-Weeds` scored z=1.19 and did not flag; under net growth it is the
clear #1 at netΔ +2,124. Live-validated 2026-06-27.)*

Scored **per `service_name`** and **per `comm_name`** *separately* (not the service×community cell):
- Over the **last 4 seven-day windows** ending at the latest record (`max(requested_date)`), weekly
  **net = inflow (`requested_date`) − closed (`closed_date`)**. **netΔ** = sum of weekly net (backlog the
  slice added over the window). Rank by netΔ.
- **Flag "growing"** when netΔ > 0 **and** the slice is net-positive in **≥3 of the 4 weeks** (sustained, not
  a one-week blip); **"watch"** at 2 of 4.
- **Small-sample:** a slice with **total inflow < 30** over the window is shown but **not ranked**.
  Null/unattributed slices and `311 Contact Us` are excluded from ranking.
- **Caveat (corrected 2026-06-27 — this file was still stating the pre-correction direction until 2026-07-02):**
  the most-recent 7-day window is under-reported on **both** sides, but inflow lags *more* than closed
  (≈0.82× vs ≈0.91× of the 4-week mean) — so week-0 net is **understated**, not overstated. A genuinely
  growing slice can lose its latest week and miss the flag. The ≥3-of-4 sustained rule absorbs most of this;
  window length (4 weeks) is a tunable.

*Day-3 upgrade candidate (NOT implemented, do not claim):* a stricter `service_name × comm_name` cell view
and calendar Mon–Sun weeks. The net-growth signal above **is** implemented and verified (eval C13).

---

## Null Community / Service Name

Rows with null `comm_name` or `service_name` → keep as **"Unknown"**, but **exclude from community/service rankings**.

Rationale: cannot triage what isn't attributed. Silently dropping them would change the headline count.

---

## API Currency Lag

API metadata "updated" stamp can lag actual latest rows. Always verify currency live:
```
SELECT MAX(requested_date) FROM iahh-g8bj
```
Show the result as the staleness stamp in the UI, not the cached metadata stamp.

---

## 311 Contact Us — non-actionable backlog

`311 Contact Us` is the **single largest open category: 26,489 rows** (35% of the ~76k open backlog).
These are informational inquiries, not field-dispatchable work orders — they cannot be surged to crews.

If this is omitted: the agent will flag `311 Contact Us` as the top surge target every week, making the
decision aid misleading. It must be filterable and excluded from crew-triage rankings by default.

Exclude via `service_name != '311 Contact Us'` in surge/aging panels, or show it separately with a
"non-actionable" label.

---

## Closed Metrics Require Closed Filter

Resolution/aging-of-closed metrics:
```
WHERE closed_date IS NOT NULL AND status_description = 'Closed'
```
Do not mix open and closed rows in resolution-time calculations.

---

## Community Quadrant Grouping — inferred label, not an official Socrata field

The Community filter dropdown groups by `:@computed_region_4a3i_ccfj` (a real Socrata point-in-polygon join
against Calgary's official NW/NE/SW/SE quadrant boundaries — no new dataset, already on `iahh-g8bj`). But
Socrata returns numeric IDs (1–4) only, with **no text label**. The `1→SW, 2→NW, 3→SE, 4→NE` mapping is an
**inferred assumption** (derived from each id-bucket's centroid position vs. downtown), live-spot-checked
against 5 known communities (Aspen Woods→SW, Tuscany→NW, Forest Lawn & Auburn Bay→SE, Rundle→NE — all
matched, 2026-07-02) but not authoritative. If Socrata ever recomputes/re-IDs this region field, the
hardcoded `QUADRANT_LABEL` map in `dashboard/index.html` could silently mislabel — re-run the Tuscany spot
check (`$where=comm_name='TUSCANY'`, expect `quad="2"`) if the grouping ever looks wrong.

**Resolved 2026-07-02 — these are not a lookup gap, they're unnamed land:** the ~52 alphanumeric placeholder
`comm_name` codes (e.g. `01B`, `02F`) were cross-referenced live against the City's own **Community District
Boundaries** dataset (`surr-xmvs`, keyed on the same `comm_code`). Result: 43 of 52 exist there, but every one
maps `name == comm_code` exactly (e.g. `02F` → `"02F"`), tagged `class="Residual Sub Area"` with
`comm_structure` UNDEVELOPED/OTHER; the remaining 9 codes don't exist in that dataset at all. Conclusion:
these are officially unnamed parcels (future-growth/industrial/reserve land not yet tied to a named
community), not a data-quality defect in `iahh-g8bj` and not resolvable via a second dataset. The dashboard
now relabels them for display as "Unassigned area (code X)" — `commLabel()` in `dashboard/index.html`, matched
via `/^[0-9]{2}[A-Za-z]$/` — while the underlying filter/query value stays the raw code, so traceability is
unaffected.

**Second use added 2026-07-03 — the service-candidate quadrant heatmap:** `loadCandidateQuadrants()` reuses
this exact same `:@computed_region_4a3i_ccfj` join and `QUADRANT_LABEL` map to show a service type's open
backlog broken down by city quadrant (see CLAUDE.md's "Quadrant-level backlog heatmap"). Same caveat applies —
if the region-ID mapping ever needs correcting, fix it once in `QUADRANT_LABEL` and both the Community filter
grouping and this heatmap update together, since they share the same lookup. Live-verified 2026-07-03: for
`Bylaw - Long Grass - Weeds Infraction`, the four quadrant counts (564+782+676+702) summed to exactly 2,724 —
the service's full actionable backlog, with zero rows falling into "Unclassified."

## Recommended-first-review escalation — a second, narrower threshold than the n<30 ranking guard

Added 2026-07-03 to resolve feedback that growth (volume) and age (wait-time) were being treated as the same
risk. `attachAgingBand()` counts, per already-surfaced growing/watch candidate, actionable open items in the
**90–365-day** band only (the same boundary the Backlog Aging panel already shows). A candidate with **≥5**
such items is pinned to the top of "Recommended first review" regardless of its growth flag.

**If this is omitted:** the recommendation silently reverts to ranking by growth/backlog size alone, and a
quietly-aging queue (netΔ ≈ 0, but a real cluster of 90–365-day-old open tickets) never surfaces as the
recommended review target — the exact gap the classmate/instructor feedback identified.

Two things not to confuse this with:
- **The n<30 small-sample guard** governs whether a slice is *ranked at all* (a statistical volume-stability
  check). This escalation threshold (N=5) is a literal headcount of real waiting tickets in one band — a
  different kind of number for a different purpose; do not conflate the two or "fix" one to match the other.
- **Historical note (superseded 2026-07-03):** this escalation trigger originally had to deliberately exclude
  a separate 1–2yr aging band, which carried its own `verifyAfterDays` "⚠ verify before dispatching" caveat
  (frequently stalled/duplicate/already-resolved-but-not-closed records) — escalating on that band would have
  told the operator "review this first" and "don't trust this" about the same records. That whole distinction
  is now moot: the aging cap was tightened to 1 year the same day (see "Aging Cap" above), so the 1–2yr band
  no longer exists as an actionable concept and `verifyAfterDays` was fully retired. The 90–365d escalation
  band itself needed zero change — it was always inside the actionable window.

## Service Category Grouping — derived from an existing naming convention, not a new field

The Service filter dropdown groups by category (e.g. Roads, Bylaw, WATS, Parks — 64 groups) by splitting
`service_name` on its first `" - "` delimiter (`categoryOf()` in `dashboard/index.html`). Live-verified
2026-07-02: 464 of 476 currently-open `service_name` values follow this "Category - Subcategory - Detail"
convention cleanly; the other 12 (e.g. `311 Contact Us`, `N/A`, `Green Line Inquiry`) have no hyphen and fall
into an "Other" group. This is a display-only derived grouping — no category field exists in the dataset, and
the `<option value>` stays the exact raw `service_name` string. Also fixed the same day: the options query's
`$limit` was 200 against 476 real distinct open `service_name` values, silently making 276 (58%) of them
unselectable — raised to 1000.

## Citywide Backlog Density Map — shipped and removed 2026-07-03, superseded by the on-demand modal map

Built earlier 2026-07-03 (`loadDensityMap()`, grouping the full actionable backlog into a `floor(latitude*100)`/
`floor(longitude*100)` grid via one SoQL aggregate query — live-verified `floor()` works where a plain
`round()` does not, `query.soql.no-such-function`) and **removed the same day** — team decision, based on real
peer feedback that a persistent citywide panel added to "too many numbers at once." The spatial need is now
served by the quadrant heatmap (unchanged) plus a new **on-demand community-map modal** (see below) — both
scoped to a specific candidate, opt-in, zero default-view footprint, which is the actual distinction the peer
feedback was about. The grid-binning technique itself remains sound and live-verified (296 populated ~1.1km
cells, 46,703 geocoded actionable items excluding 311) if a citywide aggregate view is ever wanted again.

## On-demand Community Map Modal — real boundary polygons, `surr-xmvs`

Added 2026-07-03 (evening) as a click-to-open modal on service candidates, after researching the real
`calgary.ca/311` site's own geographic views (a Power BI ward/community choropleth, plus a separate
third-party "311 live map" showing individual ticket locations by Motorola Solutions — the latter explicitly
not replicated, see CLAUDE.md's Non-goals). **Data source: `surr-xmvs`** ("Community District Boundaries"),
**not** `ab7m-fwn6` — the latter is a Socrata map-visualization view with zero queryable columns; live-verified
it returns empty `{}` rows and `no-such-column` on every real field. `comm_code` joins cleanly both directions
between `surr-xmvs` and `iahh-g8bj` (e.g. `LEB`↔`LEWISBURG`, `CSC`↔`CITYSCAPE`, live-verified). Socrata's
`simplify_preserve_topology(multipolygon, tolerance)` compresses geometry server-side — live-verified at
tolerance 0.001, all 313 communities shrink from ~2.08MB/82,678 vertices raw to ~24KB gzipped/2,890 vertices,
with no visible shape loss (a sampled Beltline polygon: 752 raw vertices → 10 at this tolerance). A plain
`round()` is unsupported on this dataset too — same `floor()`-only rule as the density-map grid above; use
`simplify_preserve_topology()`/`simplify()`, both live-verified to work, for geometry itself.

**Coverage gap** (live-verified against the current actionable backlog, not just a historical estimate): 305
of 307 comm_codes in the backlog have a matching boundary in the current 313-row snapshot. The 2 that don't —
`ABT` (a real, newer community named AMBLETON with no polygon yet) and placeholder code `09K` — total just 4
open items citywide; per-service coverage varies (e.g. for `Bylaw - Long Grass - Weeds Infraction`, live-
verified 100% match, 0 unmatched). Always disclose the unmatched count in the modal rather than silently
dropping it from the total.

**Projection:** a single equirectangular projection with one cosine-of-latitude correction at the data's
midpoint latitude is accurate to <0.5% at Calgary's city scale — this is what D3's `geoEquirectangular` does
internally, minus the Earth-radius term (irrelevant for a display-only render). No more sophisticated
projection (Mercator, Albers, UTM) buys a visible improvement at this scale and each costs meaningfully more
code. GeoJSON rings → SVG path: `"M "+points.join("L")+"Z"` per ring, all rings of a MultiPolygon in one
`<path>`'s `d` attribute, `fill-rule="evenodd"` (safer than `nonzero` for third-party data of unknown winding-
order conformance) to correctly punch holes if any community boundary ever has an interior ring.

**If the resolution constant (`*100`) is ever changed:** re-verify the cell count stays renderable (a much
finer grid, e.g. `*1000`, would return thousands of cells and need pagination or a coarser fallback) and that
`floor()` — not `round()` or `ceil()` — is the function actually used in the query.
