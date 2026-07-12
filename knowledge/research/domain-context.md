# Domain context — Calgary 311 backlog triage

> **Status:** research note (Day 1 / PLAN). Treat every claim here as **a claim to verify**, not
> settled fact (MMA 616 method, p27). External figures are cited; internal figures are recomputed
> live from the source and dated. Last refreshed against the live source: **2026-06-25**.

## What 311 is
Calgary 311 is the city's non-emergency intake line for information and service requests — potholes,
snow/ice, bylaw concerns, waste carts, signs, water issues, and ~530 other service types. A request
comes in by phone, web, app, or other channel; it is routed to the responsible department; the
citizen gets a tracking number and an expected timeline; the request is worked and eventually
closed. The open-data feed (`iahh-g8bj`) publishes one row per request, 2012→present (the live
feed actually reaches back to 2010-01-25), refreshed daily.

## The operational difficulty (why this is worth a dashboard)
The raw feed shows *individual rows*. What an operations lead cannot see from it — and what keeps
the queue from breaking — is **where work is piling up faster than it clears, and where the oldest
work has gone stale.** Three difficulties, with evidence:

1. **The backlog ages badly.**
   - *Data fact (live, 2026-06-25):* 76,264 requests open (`closed_date IS NULL AND status_description = 'Open'`); 47,338 open > 1 year; 9,972 open > 5 years; oldest open item dates to **2014**.
   - *Interpretation:* A queue with decade-old items is not being triaged by age — that signal is invisible in the raw feed.
   - *Decision use:* The aging panel surfaces this directly; items > 1 year are the escalation starting point.

2. **Inflow spikes against the seasonal norm and overwhelms a category.**
   - *Data fact (external — verify):* Q2 2024 drew more pothole complaints than any quarter since 2012 — 6,267 requests Apr 1–May 31, ~100/day ([CBC News, 2024](https://www.cbc.ca/news/canada/calgary/potholes-calgary-311-complaints-roadway-maintenance-1.7220952)).
   - *Interpretation:* A surge like this is invisible in the raw feed until it has already buried the crew.
   - *Decision use:* The spike flag (z-score vs trailing 8-week norm) surfaces onset as it begins — not in hindsight. **Limit:** a multi-week surge eventually normalizes into the window and stops flagging; the signal is onset/acceleration, not sustained duration.

3. **Response time is long and uneven.**
   - *Data fact (external — verify):* Average response time ~336 hours (~14 days), 2021–2024 ([Zibrila, Medium](https://medium.com/@zibrila.asiah/a-comprehensive-analysis-of-311-service-requests-in-calgary-e04eff5237ed)). The City's own 311 dashboards ([calgary.ca/311/dashboards](https://www.calgary.ca/311/dashboards.html)) report volumes, not where the queue is breaking.
   - *Interpretation:* The 14-day average hides the long tail — the 47k items open > 1 year are what the average masks.
   - *Decision use:* Community × service aging is the equity lens: chronic slow-service communities are undetectable without it.

## Why it matters
Crews and budget are fixed week to week. Sending them to the wrong category or community means a
real spike (a freeze-thaw pothole wave, a storm-driven snow/ice surge) sits unworked while effort
goes to a quiet queue. The cost of poor triage is aged backlog, missed expected timelines, and the
trust erosion that follows — the City's own 311 program tracks response performance for exactly this
reason ([311 summary-report reference](https://www.calgary.ca/311/summary-reports-reference.html)).

## The decision this enables
A **City of Calgary 311 / service-line operations lead**, at the **Monday standup**, makes the
**weekly crew-triage call**: *which open service types and communities do we escalate or surge crews
to this week?* The leverage a dashboard adds over the raw feed: **backlog aging** (how long the
oldest open items have waited) + **spike-vs-norm flagging** (this week's inflow ranked against its
own trailing-8-week norm).

## Objective (the no-dataset sentence)
> **This lets a Calgary 311 operations lead decide, each week, where to surge crews — by making
> visible which service types and communities are spiking above their own recent norm and where the
> oldest open work has aged — a judgment the raw request feed cannot support.**

## Known data-quality frictions (carried into `data-cautions.md`)
- `status_description` is noisy: alongside `Open`/`Closed` it includes `Duplicate (Open)`,
  `Duplicate (Closed)`, and `TO BE DELETED` — so **actionable backlog = `closed_date IS NULL AND
  status_description = 'Open'`** (both fields, not `closed_date` alone, which over-counts by ~2,300).
- `311 Contact Us` is ~26k of the open backlog — largely non-actionable inquiries, not field work;
  it is **excluded from crew-triage rankings & spikes** so it doesn't dominate (still in the headline total).
- `address` is ~100% null; location must come from `comm_name`/`comm_code`/`point`.
- The "Current Year" view (`arf6-qysm`) is rolling/capped — only the full-history id `iahh-g8bj`
  supports trend and backlog.

## Sources
- [CBC News — Calgary pothole complaints (2024)](https://www.cbc.ca/news/canada/calgary/potholes-calgary-311-complaints-roadway-maintenance-1.7220952)
- [A Comprehensive Analysis of 311 Service Requests in Calgary — Medium](https://medium.com/@zibrila.asiah/a-comprehensive-analysis-of-311-service-requests-in-calgary-e04eff5237ed)
- [City of Calgary — 311 dashboards](https://www.calgary.ca/311/dashboards.html) ·
  [311 summary-report reference](https://www.calgary.ca/311/summary-reports-reference.html) ·
  [311 public service request dashboard](https://www.calgary.ca/311/service-request-dashboard.html)
- Live source: [data.calgary.ca/resource/iahh-g8bj](https://data.calgary.ca/resource/iahh-g8bj) (internal figures recomputed 2026-06-25)
