---
name: storm-assessment
description: Produce a drainage-operations storm briefing from the live Edmonton rainfall feed. Use whenever the user asks about the current storm situation, rainfall conditions, whether to dispatch crews or barricade underpasses, what the alert level is, or wants a status readout for the duty supervisor — even if they just say "what's it looking like out there" or "give me a rundown". Fetches live data, computes the seven performance metrics, and answers in the supervisor's action framework.
---

# Storm Assessment Briefing

You are acting as the project's drainage operations analyst assistant (persona in `CLAUDE.md`).
Consult `edmonton-rainfall-metadata` skill for endpoint/field rules before querying.

## Procedure

1. **Fetch** the live snapshot:
   `https://data.edmonton.ca/resource/x4rm-mppc.json?$$exclude_system_fields=false&$limit=100`
   Parse `_24h_total` → float, `storm_rating` → int (they arrive as strings).
2. **Compute the seven metrics**:
   - Citywide alert level = max rating; name its band (0–2 Normal / 3 Monitor / 4 Dispatch / 5 Barricade / ≥6 Escalate)
   - Station counts per band, listing stations at ≥4 by name
   - Peak 24h total (mm) + station
   - Storm shape: stations ≥3 clustered (concentrate crews) vs spread out (distribute)
   - Trend, only if history is available (dashboard localStorage buffer or repeated fetches this
     session) — otherwise say trend is unknown and why
   - Gauge freshness: age of `:updated_at` (UTC → Mountain); flag > 60 min as stale
   - Data-as-of stamp — this leads the briefing
3. **Brief** in this shape:

```
As of <time> MT (<age> min ago):
ALERT: <band name> — rating <N> at <station(s)>
<one-sentence situation: shape + peak>
Actions: <band-appropriate actions, per the framework in CLAUDE.md>
Watch: <rising or near-threshold stations, or "trend unknown — single snapshot">
Blind spots: <stale/missing gauges, or "none">
```

## Rules

- Data currency comes first; a stale feed changes the advice ("verify by field report").
- Quote the feed's own `storm_rating_description` when citing a station's condition.
- If the fetch fails: report **no visibility** — never imply calm weather from missing data.
- November–April: state the feed is off-season and inactive.
- Recommendations follow the band framework; do not invent thresholds beyond it.
