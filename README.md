# TripSense

### AI-Powered Adaptive Trip Planning

> **TripSense turns itinerary planning from a static recommendation problem into an adaptive optimization problem.**

TripSense creates itineraries that balance preferences, constraints, route efficiency and changing trip conditions. When a perfect plan isn't possible, it shows you the trade-off instead of quietly breaking one of your rules.

> **Portfolio prototype** for Product Manager / Technical PM / AI PM interviews. It runs locally on a hand-written Goa dataset. **The prototype uses deterministic simulated route and context data to demonstrate the product concept.** There is no live Google Maps or routing integration, no live traffic, weather or crowd data, no bookings and no real users. Nothing here claims user research, business impact or test results.

---

## The product in one paragraph

Most itinerary generators return a static list of attractions. Users don't need recommendations; they need a day that works: the right interests, within budget, without long drives, in a sensible order, that still works when it rains or traffic is bad. TripSense treats this as an **optimization and constraint-resolution problem**:

```
preferences → hard constraints (filter) → ranking (soft preferences) → route-aware sequencing
            → changing conditions → impact assessment → minimal replan → explicit trade-offs
```

A **deterministic planner** decides what goes in the plan, enforces every limit, calculates routes and scores, and determines changes. An **optional LLM** explains decisions and turns free-text notes into structured inputs. It can never override a hard constraint.

**Target user:** young leisure travellers planning short 3–7 day trips.
**JTBD:** *"When I am planning a short trip, I want an itinerary that reflects my preferences and constraints, and I want to quickly modify it when something changes, so I don't have to manually rebuild my day."*

## Product principles

1. **Never silently violate a hard constraint to make the itinerary look complete.** Filter first, optimise what's left, surface conflicts when nothing valid remains.
2. **Optimise the day, not the list.** Sequence is part of the product: total travel, not just the next hop.
3. **Change as little as possible.** Keep what still works; replace only what the change breaks.
4. **The user decides trade-offs.** Leave time free, relax *one named limit for one activity*, or keep the original. Relaxations are always visible.
5. **Explain from data.** Every reason, score and delta comes from planner outputs, never from generated prose.
6. **AI is optional.** Everything works in deterministic demo mode.

## USP and differentiators

TripSense's USP is **not** "integrates Google Maps"; a routing API is production infrastructure, not differentiation. It is **adaptive itineraries that continuously balance preferences, constraints, route efficiency and changing trip conditions**, through:

1. **Constraint-aware planning.** Hard limits are guarantees, not weights.
2. **Route-efficient sequencing.** Each day is optimised as a route (stay → stops → stay).
3. **Context-aware replanning.** Traffic, crowding and weather (simulated) trigger an impact check and a minimal replan.
4. **Explicit trade-off resolution.** "No suitable replacement within your travel limit" plus clear choices.
5. **User-controlled adaptation.** Nothing destructive is applied without Apply.

## Demo walkthrough (about 5 minutes)

1. **Landing** → *Plan my trip*.
2. **Setup:** stay in *South Goa*, 4 days, ₹4,000/day, *Balanced*, interests *Beaches, Food, History*, turn on **Avoid long travel**. The live constraint check drops to ~14 / 60 eligible.
3. **Generate.** The steps shown are the real pipeline: filter → score → schedule → optimise route → summary.
4. **Trade-off detected:** the limits fill ~12 of 16 planned stops, so TripSense recommends leaving ~7 h free. Show *Keep 12 activities* vs *Relax travel limit · fills N slots*.
5. **Itinerary:** open **Why this activity?** (planner signals, your limits, + / − trade-off, score breakdown). Inspect **Smart Route** (minutes and km per leg, totals, free time, route savings) and **Trip Efficiency** (*How is this calculated?*).
6. **Adaptive Day:** on Day 2 set *Traffic → Heavy*. Transfers turn red, Trip Efficiency drops, and *N activities affected, some now break your limits* appears. Click **Optimize My Day**.
7. **Proposal:** affected activities, before/after, travel / distance / efficiency / constraint-fit deltas, and a **Trade-off · your decision** card for each slot with no valid option. *Apply* stays disabled until each is decided. Choose *Leave this time free* for one, *Relax travel limit* for another, then **Apply**. The relaxed activity is marked **Travel preference relaxed**.
8. **Skip & swap** an activity on Day 1. With the 35-min limit there is no valid replacement, so you get **"No suitable replacement within your travel limit"** instead of a silent long transfer. Choose *Relax travel limit* and confirm.
9. **Replan My Day** with the note *"It's raining heavily and I don't want to travel more than 20 minutes."* The note becomes rain + a 20-min cap, and the planner enforces it.
10. **Insights:** core and adaptive-planning metrics from your own session's events.

## Run it

Requires Node 20+ (tested on Node 24).

```bash
npm install
npm run dev            # http://localhost:3000
```

Optional, to enable live LLM text (the app works fully without it):

```bash
cp .env.example .env.local
# set ANTHROPIC_API_KEY=...   (read server-side only)
```

Quality gates:

```bash
npm test               # planner + adaptive suites (deterministic, no network)
npm run typecheck
npm run lint
npm run build && npm start
```

## What's real vs simulated

| Area | Status |
|---|---|
| Constraint filtering, scoring, route sequencing, replanning, adaptive replanning, conflicts | **Real.** Deterministic TypeScript, tested |
| Trip Efficiency, Smart Route totals, before/after deltas | **Real computation** over simulated inputs |
| "Why this activity?" and "Why did TripSense change this?" | **Real.** Derived from planner attributes and the computed diff |
| Activity data (60 Goa activities: costs, durations, crowds, popularity, visiting times) | **Simulated.** Hand-written demo data |
| Coordinates, distances, travel times | **Simulated.** Demo coordinates; haversine × 1.3 road factor; speed by distance |
| Traffic, crowding, weather | **Simulated.** Set by the user ("demo context, not live data") |
| AI summaries, note interpretation, change explanations | **Real LLM** when `ANTHROPIC_API_KEY` is set. Otherwise **deterministic demo mode**, labelled in the UI |
| Analytics & metrics | **Real instrumentation**, local data only. Not user research |

## AI vs deterministic responsibilities

| Deterministic planner (source of truth) | LLM (optional) |
|---|---|
| Decides which activities are in the plan | Writes the trip summary |
| Enforces every hard limit (budget, travel, transfers, outdoor, crowds, early starts) | Interprets free-text notes into a **fixed schema** (triggers, interests, travel cap), validated and sanitised |
| Calculates routes, distances, times, Trip Efficiency | Explains a diff the planner already computed |
| Assesses context impacts, proposes minimal changes, builds conflicts | Never selects activities, never decides whether a limit is broken |

If the LLM is unavailable or returns invalid output, every surface falls back to templates or keyword matching, and invalid travel caps (outside 10–120 min) are ignored.

## Architecture at a glance

```
Browser                                                   Server (Next.js route handlers)
───────                                                   ──────────────────────────────
React UI ── planner (pure TS) ── Goa dataset + coords      /api/ai/summary   ─┐
   │         ├ constraints → scoring → schedule/optimise   /api/ai/interpret ─┼─ provider.ts ── Anthropic API
   │         ├ route model · evaluate (efficiency, audit)   /api/ai/explain   ─┘     │
   │         └ replan · adaptive · conflicts               /api/ai/status            └─ fallback.ts
   ├── localStorage (trip v2, events)
   └── ai/client.ts ── falls back locally
```

- **Stack:** Next.js 15 (App Router), React 19, TypeScript (strict), Tailwind CSS v4, `@anthropic-ai/sdk`. No other runtime dependencies.
- Full detail, diagrams and formulas: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Documentation

| Doc | Contents |
|---|---|
| [docs/PRD.md](docs/PRD.md) | Problem, users, JTBD, principles, requirements (incl. adaptive & conflict), prioritisation, risks |
| [docs/PRODUCT_DECISIONS.md](docs/PRODUCT_DECISIONS.md) | Why local data, deterministic ranking, hard vs soft, route optimisation approach, conflicts-as-policy, simulated context, LLM placement |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Mermaid diagrams, constraint model, route model, sequencing, Trip Efficiency, adaptive replanning, AI layer, tests |
| [docs/METRICS.md](docs/METRICS.md) | Metric tree, definitions incl. adaptive metrics, event dictionary |
| [docs/EXPERIMENTS.md](docs/EXPERIMENTS.md) | Proposed A/B tests: explanations → acceptance; explicit trade-offs vs auto-fill |
| [Documents/case-study.html](Documents/case-study.html) | Portfolio case study |

## Product metrics (definitions only, no results)

- **Primary: Activity Acceptance Rate.**
- **Core:** Trip Generation Completion · Constraint Violation Rate (guardrail, must be 0) · Time to Generate.
- **Adaptive planning:** Average Trip Efficiency · Average Daily Travel Time / Distance · Replan Usage · Adaptive Replan Acceptance · Constraint Conflict Rate · Constraint Relaxation Rate · "Leave Time Free" Rate · Activity Acceptance Rate after Replan.

Definitions are in [docs/METRICS.md](docs/METRICS.md). `/insights` computes them from your own session.

## Interview Talking Points

### The main story: from recommendation to optimization

> *"I initially approached itinerary generation as a recommendation problem. While testing edge cases, I found that optimizing for itinerary completeness could cause the system to violate explicit travel constraints. That led me to treat trip planning as an optimization and constraint-resolution problem instead."*

The concrete moment: skipping an activity sometimes produced a replacement **~65 minutes away**. The first instinct was to tune weights (penalise distance more), and I did. It reduced the symptom but didn't fix the cause: the planner still *preferred a complete day over the user's limits*. Fixing it properly was a **product policy** decision:

1. User gives preferences and constraints, including an always-on transfer limit.
2. Planner **filters** hard constraints. They are never weights.
3. **Route-aware optimisation** builds each day (exact search over order and slots for minimum total travel).
4. **Context changes** (simulated traffic, crowding, weather) can invalidate the plan.
5. TripSense **evaluates affected activities** and separates "now breaks a limit" from "now worse".
6. It proposes a **minimal replan**: unaffected activities stay; replacements must be valid and nearby.
7. If constraints conflict, it **surfaces the trade-off**: no silent long transfer.
8. **The user controls** whether to leave time free, relax one limit for one activity (clearly marked), or keep the original.

### Other talking points

**Problem framing.** Generation is commoditised. The unmet need is a plan that respects *my* limits and survives contact with reality.

**MVP prioritisation.** P0: hard-limit filtering, ranking, route sequencing, explanations, replanning, conflict resolution. Cut: booking, live maps, accounts, multi-city.

**Constraint modelling.** Hard limits filter before scoring, so a high score can never outvote a limit. Trust is asymmetric: one rainy-day beach or 65-minute drive costs more trust than a slightly weaker interest match.

**Route optimisation.** Not nearest-next (locally optimal, globally zig-zag). Each day is a small TSP with time windows: exact branch-and-bound over visit order *and* slot choice, minimising travel incl. the return home plus a small penalty for leaving an activity's best time. A unit test shows a case where nearest-next loses.

**Explainable scoring.** Trip Efficiency = preference fit, constraint fit, route efficiency and time utilisation, each with a stated formula and computed from the plan. Not an AI score.

**AI product design.** The LLM sits at the edges: notes become a validated schema, and computed diffs become prose. Because it never chooses activities, plan validity doesn't depend on model behaviour.

**Metrics.** Primary: Activity Acceptance Rate. Guardrail: Constraint Violation Rate = 0 (tested and self-checked at runtime). New adaptive metrics answer *"do users accept adaptive changes?"* and *"how often do limits collide, and what do people choose?"* Relaxation Rate vs Leave-Time-Free Rate is a direct read on how users value limits vs completeness.

**What I'd validate first.** Whether travellers prefer an honest gap over a filled-but-worse day (see Experiment 2 in docs/EXPERIMENTS.md).

## From prototype to production

| | V1 prototype (today) | Production |
|---|---|---|
| Places | 60 hand-written Goa activities | Places API / licensed + curated place data, freshness checks |
| Routes | Demo coordinates, haversine × road factor | Google Maps Routes API or equivalent routing provider |
| Traffic | Simulated (user-set, × 1.6) | Live / predicted traffic |
| Weather | Simulated (user-set) | Real forecast; proactive replan suggestions |
| Crowds | Simulated level shift | Real crowd / availability signals where legally & technically available |
| Opening hours | Typical time-of-day slots (demo) | Real opening hours & closures |
| Planner | Deterministic, in-browser | Same logic as a service, with monitoring |
| LLM | Optional explanations / interpretation | Evals for interpretation accuracy, cost & latency monitoring |
| Persistence | localStorage | Accounts, cloud persistence, sharing |
| Learning | Hand-tuned weights | Personalised learning from keep/skip and relax/leave-free behaviour |
| Experimentation | Proposed only | Assignment service, analytics pipeline (sink adapter exists) |

## Limitations

- Travel model is straight-line × factor; real roads, ferries and one-way systems are not modelled.
- Simulated traffic is a uniform multiplier; no time-of-day profiles.
- Single destination; data is approximate and static.
- Sequencing is exact only for small days (≤ ~7 stops, with a node cap).
- "Keep original" in a conflict is recorded as an explicit relaxation; there is no undo history beyond the last applied change per day.
- Local-only analytics; no identity, no real users.

## Project structure

```
src/
  app/                 # pages: / , /plan, /trip, /insights; api/ai/* routes
  components/trip/     # timeline, Smart Route, Efficiency, Adaptive Day, proposals, conflicts, Why drawer
  lib/
    data/              # destination datasets (demo data + coordinates) + registry
    planner/           # constraints, scoring, route, schedule (optimiser), evaluate, generate, replan
    ai/                # provider (server), client (browser), fallback (shared)
    analytics.ts       # typed event tracking with pluggable sinks
    metrics.ts         # metric calculations for /insights
scripts/
  verify-planner.ts    # original planner invariants
  verify-adaptive.ts   # route, conflict, adaptive & explanation tests
docs/                  # PRD, decisions, architecture, metrics, experiments
Documents/             # case study
```
