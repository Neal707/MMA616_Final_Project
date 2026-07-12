# Calgary 311 — Domain Research

*Audience: business stakeholders and team members unfamiliar with municipal service operations.*
*Scope: background needed to understand why the Backlog Triage dashboard matters and what decisions it supports.*

---

## 1. Purpose of 311

Calgary 311 is the City's single front door for all non-emergency, non-police municipal services. Launched in 2004 and celebrating 20+ years of operation, it consolidates what was previously a fragmented set of departmental phone numbers into one contact point available **24 hours a day, 365 days a year** by phone, web form, and mobile app.

Its core promise: any resident, business owner, or visitor can report a problem or ask a service question once, receive a tracking number, and trust that the right work group will be notified automatically. 311 does not resolve requests itself — it **receives, classifies, routes, and tracks** them on behalf of the operating departments.

---

## 2. Who Uses It

| User group | Why they use 311 |
|---|---|
| **Residents** | Report potholes, overgrown weeds, missed waste pickup, graffiti, damaged trees, bylaw concerns |
| **Business owners** | Report sidewalk/lane issues, request permits, access tax and business license information |
| **Visitors** | Navigate city services without knowing which department to call |
| **City departments** | Receive structured, attributed work orders; generate performance reports |
| **Council / Ward offices** | Monitor complaint volumes and response times by ward for constituent accountability |
| **Operations managers** | Review the queue of open requests to plan crew assignments for the week |

Between January 2021 and May 2026, Calgary received over 2.6 million 311 service requests — roughly 500,000 per year. Demand follows a strong seasonal cycle: it peaks in summer (vegetation, infrastructure wear) and dips in winter, though snow-and-ice requests create secondary spikes.

---

## 3. How Requests Flow Through the System

```
Citizen contact  →  311 intake  →  Classification  →  Work order created
                                                              ↓
                                                    Assigned to department
                                                              ↓
                                              Crew dispatched / scheduled
                                                              ↓
                                                    Work completed → Closed
                                                    (or deferred → Backlog)
```

**Step by step:**

1. **Intake** — Citizen contacts 311 by phone, online form, or mobile app. The agent (human or automated) captures the request type, location, and any relevant details.
2. **Classification** — The system maps the request to a `service_name` category (e.g., "Bylaw — Long Grass and Weeds") and an `agency_responsible` (the operating department).
3. **Work order creation** — A structured record is created with a tracking number, timestamp, community, and status `Open`. The citizen receives the tracking number for follow-up.
4. **Departmental routing** — The work order lands in the assigned department's queue. If the issue can be resolved by information alone, 311 closes it immediately. Physical work orders stay open until a crew acts.
5. **Resolution or deferral** — A crew resolves the issue and closes the record (status → `Closed`). If resources are unavailable, the request stays open and ages in the backlog.
6. **Duplicate handling** — Repeated reports for the same issue are flagged `Duplicate` and linked to the parent request to avoid double-dispatching.

---

## 4. City Departments Involved

The Calgary 311 dataset attributes each request to an `agency_responsible`. The principal operational departments are:

| Department | Primary request types |
|---|---|
| **Transportation — Roads** | Potholes, pavement, lane/alley repairs, street signs |
| **Parks** | Tree damage, grass/weed complaints, park maintenance |
| **Community Standards (Bylaw)** | Noise, property standards, long grass, unsightly premises, animal issues |
| **Waste & Recycling** | Cart management, missed collection, bulk waste |
| **Utilities & Environmental Protection** | Water/sewer issues, drainage, spills |
| **Calgary Transit** | Shelter damage, stop/signage issues |
| **Corporate Properties** | City-owned building maintenance |

Waste-related services (cart management, missed collection) historically generate the largest single-category volumes. Vegetation complaints (long grass, trees) are the dominant summer driver. Snow and ice management peaks in winter.

---

## 5. Why Backlog Management Matters

A **backlog** is the set of open service requests that have been received but not yet resolved. It grows when inflow exceeds crew capacity and shrinks when crews clear more than arrives. Backlog management is one of the most operationally consequential functions in a 311-driven city, for three reasons:

**Service level commitments.** Most request types carry a target response time (e.g., pothole patched within X days of report). Requests that sit unattended breach those commitments and generate complaints, repeat reports (which inflate the apparent backlog), and political pressure.

**Budget and resource impact.** Nearly 40 percent of municipal operating budgets in North American cities are consumed by reactive repairs to deferred maintenance. A backlog that is allowed to age converts manageable service requests into more expensive interventions — a pothole that becomes a pavement failure, a drainage issue that floods a basement.

**Equity and trust.** Backlogs do not distribute evenly across communities. If some neighbourhoods consistently wait longer, the pattern becomes visible (and politically significant) once it is measured. Transparent, data-driven triage is the primary tool for demonstrating equitable service delivery.

---

## 6. How Operations Managers Allocate Crews

Each week, a 311/service-line operations lead faces a planning question with real constraints:

- **Fixed crew capacity** — field crews are finite; overtime is expensive.
- **Variable inflow** — this week's new requests may cluster around a storm event, a seasonal transition, or a community-specific trigger.
- **Aging backlog** — older open requests have priority claims on crew time, especially if they are approaching SLA breach.
- **Geography** — sending a crew to one end of the city for two requests when ten are clustered nearby is inefficient.

Without a dashboard, the manager relies on departmental spreadsheets, phone calls, or raw database queries to reconstruct the picture. The weekly planning meeting ("where do we put crews?") typically covers:

1. How large is the current open backlog, and how does it compare to last week?
2. Which categories and communities generated the most new requests this week — is it a spike or normal variation?
3. Which open requests are the oldest — which communities or service types are most at risk of SLA breach?
4. Are there geographic clusters that would make a crew run efficient?

These are the exact four questions the dashboard is designed to answer in one view, without requiring the manager to write a query.

---

## 7. What Decisions Our Dashboard Supports

**Decision:** *Which open service types and communities should receive crew surge/escalation this week?*

**Context in which it is made:** Monday standup, operations lead, 10–20 minutes available before crews are dispatched.

**How the dashboard supports it:**

| Dashboard component | Decision it informs |
|---|---|
| **Headline backlog count + status mix** | Is the overall queue growing or shrinking? Is triage working? |
| **This-week inflow vs. trailing-norm** | Is this week's volume a real spike or normal variation? Which categories are above their norm? |
| **Backlog aging table** | Which communities and service types have the oldest unresolved requests? Where is breach risk highest? |
| **Category × community breakdown** | Where are requests concentrating? Are there actionable geographic clusters? |
| **Staleness stamp ("data as of …")** | Can I trust these numbers, or is the source stale? |

**What the dashboard does not decide:** it does not tell the manager *how many* crews to send, predict future volumes, or rank individual requests for a crew route. Those decisions remain with the human — the dashboard's job is to make the weekly picture clear enough that the call can be made in the room.

---

## Sources

- [311 Calgary](https://www.calgary.ca/311.html)
- [311 Calgary FAQ](https://www.calgary.ca/311/faqs.html)
- [311 Service Request Summary Reports Reference](https://www.calgary.ca/311/summary-reports-reference.html)
- [311 Service Requests — Open Calgary (dataset iahh-g8bj)](https://data.calgary.ca/Services-and-Amenities/311-Service-Requests/iahh-g8bj)
- [311 Celebrates 10 Years of Citizen Services](https://newsroom.calgary.ca/311-celebrates-10-years-of-citizen-services/)
- [Exploring Calgary's 311 Service Requests — Medium](https://medium.com/@zhanayla/exploring-calgarys-311-service-requests-unveiling-urban-insights-b406f26cb123)
- [What Do Calgary Residents Ask for Most? — Medium](https://medium.com/@lcc9316/what-do-calgary-residents-ask-for-most-a-data-story-of-311-service-requests-e7088c58ff9b)
- [Municipal Work Order Management: From Citizen Request to Resolution](https://oxmaint.com/industries/government/municipal-work-order-management-citizen-request-resolution)
- [What is a 311 CRM Solution? — CivicPlus](https://www.civicplus.com/blog/crm/what-is-a-311-and-citizen-request-management-solution/)
