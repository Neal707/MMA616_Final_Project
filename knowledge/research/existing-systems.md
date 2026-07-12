# Existing systems & our differentiation

> Research note (Day 1). Claims to verify, sources cited. The point: Calgary 311 is a **mature
> stack** — our value is not "they have no data," it's the **missing operational decision layer**.

## The real problem, sharpened
The City of Calgary does **not** lack 311 data or dashboards. What's missing is the **weekly
operational decision layer**: the ops lead's "where do we surge crews this week?" call has no
purpose-built tool. They sit between a **transactional CRM** (great for one request) and
**public volume dashboards** (built for residents and council) — neither ranks *where the queue is
breaking*. That gap is the opportunity.

## What the City already runs (and why each doesn't close the gap)

**1. System of record — Motorola PremierOne CSR.**
Calgary launched Canada's first 311 in 2005 on Motorola Solutions' PremierOne™ CSR (cloud CRM).
It does intake (phone/web/app), classification, **routing to departments, work-order creation,
status tracking, and per-request SLA**. It is **transactional** — optimized for handling an
individual request end-to-end, internal-only — not for cross-request weekly analytics like
"which category is spiking vs its own norm" or "where is the oldest work aging." It is the *source*
the open feed is published from, not a triage view.
[Motorola case study](https://www.motorolasolutions.com/content/dam/msi/docs/products/smart-public-safety-solutions/citizen-engagement/premierone-csr-calgary-case-study.pdf)

**2. Public open dashboards** ([calgary.ca/311/dashboards](https://www.calgary.ca/311/dashboards.html)) — four of them:
| Dashboard | Shows | From | Audience |
|---|---|---|---|
| Public Service Requests | volume by ward, community, business unit, request type | 2016, 2h refresh | public |
| Public Information Inquiries | inquiries by topic, ward, community | 2016 | public |
| Ward Dashboard | request volume by ward, per-household, vs city avg | 2018 | public/council |
| Assessment & Tax SR · Snow & Ice | tax / snow-ice request volume by ward | 2004/2016 | public |

The City's **own framing**: these dashboards *"serve informational purposes rather than operational
management."* All are public, volume/trend reporting. **None** provide backlog aging, spike-vs-norm
deviation, SLA-cascade, or a crew-triage ranking.

**3. Open-data feed — Socrata `iahh-g8bj`.** Full history, daily, published downstream of PremierOne.
Raw rows, no decision layer — the input we build on.

## The gap none of them fills
- **Backlog aging** — oldest open by service tier / community (not in any public dashboard).
- **Spike-vs-norm flagging** — this week vs the category's *own* history (public dashboards show raw
  volume, never deviation from a baseline).
- **SLA cascade / displacement** — what a surge starves (alley/minor-road work deferred).
- **One weekly triage ranking** purpose-built for the ops lead's Monday call.

## How we leverage their systems (complement, not replace)
- **We sit on the City's own open-data feed** (already published from PremierOne) → **zero new
  integration, no procurement, no internal-CRM access required.** Reproducible and auditable.
- **We complement, don't compete:** PremierOne stays the system of record; the public dashboards
  stay for residents/council; ours is the **internal decision aid** layered on top.
- **Same data the public sees** → transparent and defensible; the prioritization could feed back to
  crews and even enrich the public dashboards.

## Differentiation in one line
> The City has a **system of record** (PremierOne) and **public reporting** (4 dashboards). We add
> the **operational decision layer** — spike-vs-norm + aging + SLA cascade — on their own open feed.

## Honest boundary (steelman the counter)
PremierOne and internal BI may include operational reports we can't see from outside. Our claim is
scoped to **what's publicly verifiable + the open feed**: an open, reproducible spike + aging + SLA
triage view does not exist in Calgary's public 311 ecosystem today. If an internal equivalent exists,
our edge is openness, reproducibility, and the SLA-cascade lens.

## Sources
- [calgary.ca/311/dashboards](https://www.calgary.ca/311/dashboards.html) ·
  [public service-request dashboard](https://www.calgary.ca/311/service-request-dashboard.html) ·
  [snow & ice dashboard](https://www.calgary.ca/311/snow-ice-dashboard.html)
- [Motorola PremierOne CSR — Calgary case study](https://www.motorolasolutions.com/content/dam/msi/docs/products/smart-public-safety-solutions/citizen-engagement/premierone-csr-calgary-case-study.pdf)
- Open source: [data.calgary.ca/resource/iahh-g8bj](https://data.calgary.ca/resource/iahh-g8bj)
