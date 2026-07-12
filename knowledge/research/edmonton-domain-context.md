# Domain context — Edmonton storm-event drainage operations

> **Status:** research note. Internal figures verified live against the source and dated; treat
> external claims as claims to verify. Last refreshed against the live source: **2026-07-12**.

## What the rain-gauge network is
The City of Edmonton (drainage services operated by **EPCOR** since 2017) maintains a network of
~23 tipping-bucket rain gauges (`RG17`–`RG62`) across the city. Each gauge reports its rolling
24-hour rainfall total (mm) and a **storm severity rating** on a 0–8 scale anchored to storm-sewer
capacity — the descriptions are written in drainage language ("sewers filling", "sewers may back
up", "underpasses and low areas may start to flood"). The open-data feed (`x4rm-mppc`) publishes
one row per gauge, refreshed every 15 minutes, May–October.

## The operational difficulty (why this is worth a dashboard)
The raw feed is 23 independent point readings. What a duty supervisor cannot see from raw rows:

1. **Where the system is stressed *right now*, ranked.** A flat JSON list doesn't order stations
   by severity, band them into action thresholds, or show which readings changed since last check.
   *Live example (2026-07-11):* stations ranged from rating 0 to rating 6 ("Flooding widespread")
   simultaneously — 68.8 mm at RG17 vs <5 mm elsewhere. The storm was highly localized; the raw
   feed doesn't say that, a map does.
2. **Whether conditions are building or easing.** The feed keeps no history — each refresh
   overwrites the last. Rising vs falling is *the* dispatch-vs-stand-down signal, and it is
   invisible without accumulating snapshots.
3. **What is a blind spot vs what is dry.** Gauges only transmit after 2 mm accumulates and
   readings are gauge-location-only. A silent gauge in an active storm is missing data, not calm.

## Why it matters
During a convective summer storm, crews, barricades, and pump checks are finite and travel time is
real. Sending crews to the wrong quadrant while a localized cell drops 40+ mm on two neighbourhoods
means flooded underpasses (a life-safety risk — vehicles stall in underpass flooding), sewer
backups into basements, and claims against the utility. Edmonton's July 2004 storm (>100 mm in
places) and repeated summer flash-flood events are the design context for the rating scale itself.

## The decision this enables
An **EPCOR / City of Edmonton drainage operations duty supervisor**, during an active rain event,
makes the **next-30-minutes call**: *where do I send crews, what do I barricade, whom do I notify,
and what do I stand down?* The leverage the dashboard adds over the raw feed: **severity ranking +
action-band counts** (how many stations at dispatch/barricade/escalate thresholds), **spatial
spread** (localized cell vs citywide), and **session-buffered trend** (building vs easing).

## Objective (the no-dataset sentence)
> **This lets a drainage operations duty supervisor decide, during an active storm, where to send
> crews, which underpasses to barricade, and when to escalate to emergency management — by making
> visible which gauges have crossed action thresholds, whether the storm is localized or citywide,
> and whether it is building or easing — a judgment 23 raw point readings cannot support.**

## Known data-quality frictions (carried into `edmonton-rainfall-library.md`)
- Snapshot-only feed → trend requires client-side accumulation; never claim official history.
- `date_time` resets to midnight daily → freshness comes from Socrata `:updated_at` (UTC).
- Numerics arrive as strings → parse before sort/math.
- Silence ≠ zero (2 mm transmit threshold; gauge-location-only readings; May–Oct season).

## Sources
- Live source: [data.edmonton.ca — x4rm-mppc](https://data.edmonton.ca/d/x4rm-mppc) (metadata + rows verified 2026-07-12)
- EPCOR — [Flood mitigation / stormwater](https://www.epcor.com/products-services/drainage) (operator context — verify)
- City of Edmonton — [Flood mitigation program](https://www.edmonton.ca/city_government/utilities/flood-mitigation) (verify)
