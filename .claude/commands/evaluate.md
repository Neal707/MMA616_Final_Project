---
description: Pre-ship evaluation workflow — verify the dashboard, run the T1–T8 test set against the live feed, run the critic, and print a pass/fail scorecard.
---

# /evaluate — pre-ship evaluation workflow

A repeatable, multi-step workflow that strings the workspace's evaluation pieces together into
one gate. Run it after any dashboard change and before a demo or submission. It tests the
**output** (are the numbers recomputable, is the narrative honest?), not the code. Any FAIL
blocks ship.

Follow these steps in order and do not skip any:

## Step 1 — Serve & open
Invoke the `dashboard-verify` skill's serve step: start the `dashboard` launch config
(python http.server on 8643) and open `http://localhost:8643`. If a screenshot times out,
use `get_page_text` / `javascript_tool` — DOM checks are authoritative.

## Step 2 — Run the dashboard-verify suite
Invoke the `dashboard-verify` skill and work its full checklist (row/marker parity, KPI
cross-checks, freshness stamp, trend buffer, failure path, filters, tabs, EPCOR layer, Stats
charts, 311 open-only, modal). Record each section pass/fail.

## Step 3 — Run the independent test set (recompute, don't trust the UI)
For each, recompute from the live source with an independent SoQL/curl query and compare to
what the dashboard shows. Report PASS/FAIL + the two numbers.

| # | Test | Independent check |
|---|------|-------------------|
| T1 | Citywide alert level | `$select=storm_rating,count(*)&$group=storm_rating` → max band == alert bar |
| T2 | Peak / worst gauge | client-side numeric `max(_24h_total)` (API `max()` is a string max) == worst-gauge tile |
| T3 | Band counts / stations ≥4 | grouped query counts == tiles + priorities; sum == station count |
| T4 | Freshness stamp | newest `:updated_at` (via `$$exclude_system_fields=false`), UTC→MT == header stamp; >60 min flagged |
| T5 | Trend honesty | inject a synthetic ≥2-sample buffer → arrows appear + "N samples over X min"; remove after |
| T6 | Blind-spot handling | stub `fetch` to reject → "DATA UNAVAILABLE … do not assume no rain"; malformed value → "no reading", never 0 mm |
| T7 | Briefing narrative | `storm-assessment`: leads with data-as-of, quotes feed wording, actions match bands, no invented history (human-reviewed, not model-scored) |
| T8 | Stats cross-check | one heatmap/chart cell == `q7ua-agfg` count for the same area/range/category incl. `WEATHER311_WHERE` |

## Step 4 — Run the critic
Launch the `dashboard-critic` subagent (it judges from the duty supervisor's chair and
recomputes any number it critiques). Triage its findings: wrong-decision risk first.

## Step 5 — Scorecard
Print a table: each check → PASS / FAIL (+ the evidence numbers), the total (n/N), and any
FAIL as a blocking item. State the data-as-of stamp the run was measured against.
