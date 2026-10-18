# Sari-Daari — Architecture

> *Sari daari* (ಸರಿ ದಾರಿ): "the right way". For each trip it answers one question:
> **metro, bus, or your own bike/car, and how do I get to the station or stop?**

Status: design, revision 2 (2026-10-18). Nothing below is implemented yet.

**Revision 2:** buses are now in the first version; added nearby stops and the right stop to walk to;
the audience is everyone living in Bengaluru (no login needed, Kannada + English); transit routing
moves to OpenTripPlanner.

---

## 1. Who it's for and what it does

**Who:** anyone living in Bengaluru, on their phone. Works without signing in. Kannada + English
(the BMTC feed already has Kannada stop names).

Google Maps already shows live traffic, so copying that is not the product. The product is a
**decision for each trip, plus the exact place to walk to**:

> *(made-up example: numbers, routes and gates are only there to show the layout)*
>
> **Koramangala 5th Block → Whitefield (ITPL), now, 8:50 am**
> 🚇 **Auto + metro: 48–55 min · ₹60** ← *recommended*
>   Auto 12 min to **Indiranagar metro (Gate B)** → Purple Line, 24 min → walk 6 min
> 🚌 **Bus: 75–95 min · ₹30** (route X from Forum Mall stop, every 10 min. Same traffic as your bike)
> 🏍 **Your bike: 68 min · ₹45 fuel** (normally 38. Traffic is 1.8× normal on ORR)
> 💡 Leave after 10:40 and the bike takes 40 min.

| Feature | Version |
|---|---|
| Door-to-door comparison: **metro, bus, bike, car**, and combinations (walk/auto + metro, bus + metro) | first version |
| **Nearby stops**: closest metro stations (by entrance) and bus stops, with walking time, lines/routes served, how often they run | first version |
| **The right stop, not just the nearest**: the stop or station the best trip actually starts from | first version |
| Live traffic on the road route + 2 alternate routes; peak-hour warning | first version |
| Saved commutes (home ↔ office), works without an account | first version |
| Leave-time advice, push alerts before you usually leave | v1 |
| Rain adjustment, metro crowding hint, "how long did it take?" feedback | v2 |
| Live bus positions | only with official BMTC data access (see §8) |

Out of scope: turn-by-turn navigation (hand off to the phone's maps app), ticketing (Namma Yatri, BMRCL and
Namma BMTC apps already do it).

---

## 2. System overview

```mermaid
flowchart LR
  subgraph Client["web/ — Next.js PWA"]
    UI[Plan + Nearby screens<br/>MapLibre map]
    SW[Service worker<br/>web push]
  end

  subgraph API["backend/ — FastAPI (Python)"]
    R[HTTP routes /v1]
    PL[Trip planner<br/>combines everything]
    BUS[Bus time correction<br/>traffic-aware]
    NB[Nearby stops<br/>in-memory KD-tree]
    REC[Recommender<br/>deterministic rules]
    LT[Leave-time advisor]
    QG[Quota guard + cache]
    W[Commute watcher<br/>separate process]
  end

  OTP[OpenTripPlanner 2<br/>Java service<br/>walk + bus + metro]
  TT[TomTom Routing,<br/>Search, Traffic tiles]
  DB[(PostgreSQL 16)]
  FEEDS[[GTFS: BMRCL + BMTC<br/>OSM Bengaluru extract]]

  UI -->|REST| R
  R --> PL --> REC
  R --> NB
  R --> LT
  PL -->|transit options| OTP
  PL -->|bike/car options| QG --> TT
  PL --> BUS --> QG
  LT --> QG
  W --> PL
  W -->|alert| SW
  FEEDS -->|nightly graph build| OTP
  FEEDS -->|load at startup| NB
  PL --> DB
  W --> DB
  UI -->|traffic tiles via proxy| R
```

Request flow for `POST /v1/plan`:

1. **Geocode** if given text: TomTom Search limited to the Bengaluru area, cached for 30 days.
2. Run in parallel:
   - **Transit:** ask OTP for up to 5 options (`WALK,BUS,SUBWAY`, plus "auto to station" via OTP's
     car-to-station mode, with walking limited to about 1.5 km).
   - **Road:** TomTom `calculateRoute`, `traffic=true`, `maxAlternatives=2`, for each vehicle the user has.
3. **Fix the bus times** (§3.3). OTP's bus times come from schedules, and buses sit in the same traffic as cars.
4. Add up cost, attach the nearest station entrance for each boarding point, **score and recommend** (§3.5).
5. Store an observation row (that's how history builds up), then return.

---

## 3. Components

### 3.1 Transit routing: OpenTripPlanner 2 (separate Java service)
Revision 1 planned a small router written by hand, which was fine for 80 metro stations. With buses the feed is
**~10k stops, 4.4k routes, 58k trips, 1.5M stop times**, and planning on it needs a proper
transit algorithm (RAPTOR) plus walking over real streets. OTP2 has both, is proven, and reads
our feeds directly:

- Graph inputs: `bmrcl.zip` + `bmtc.zip` (Vonter, ODbL) + a Bengaluru OSM extract
  (Geofabrik `india/southern-zone`, cut to the city's bounding box with `osmium`).
- Rebuild the graph nightly/weekly, then **swap it in without downtime** (start the new graph, switch the port, stop the old one).
- Our backend only talks to it through a `TransitRouter` interface, so OTP can be replaced
  (e.g. by a Python RAPTOR) without touching the planner. Tests use a `FakeTransitRouter`.
- Java 21+ (Java 25 is already installed on the dev Mac). Native jar, no Docker.
- **Memory:** plan for 3–4 GB to serve and more while building the graph. Measure once it's built. This sets the hosting cost (§6).

This doesn't conflict with "backend in Python". OTP is a separate service like Postgres, and all our logic
stays in FastAPI.

### 3.2 Nearby stops + the right stop
- At startup, load stops from both feeds: metro **entrances** (`location_type=2`), metro stations, and bus stops
  (BMTC lists each direction as its own stop, e.g. *"… (Towards AIT College)"*, so we show the direction).
- Build an in-memory KD-tree with `scipy.spatial.cKDTree` on projected coordinates. ~10k points, microseconds per query, no PostGIS.
- `GET /v1/nearby`: the k nearest **metro stations** (distance to the closest entrance) and **bus stops**,
  each with straight-line distance → walking time (× 1.3 detour factor, real walking route from OTP on tap),
  the routes/lines that stop there, and how often they run in the current hour (from `stop_times` grouped by route).
- **Right stop vs. nearest stop:** the nearest stop may not have a bus to where you're going. The plan shows
  the stop the best option actually boards at ("walk 4 min past the nearer stop to Forum Mall, where 500D stops").

### 3.3 Bus time correction (traffic-aware buses)
BMTC schedules are known to be wrong (the feed's author warns about this), and a bus sits in the same jam
as a car. So for every bus leg:

```
bus_ride = schedule_time(board→alight) × road_delay_ratio(board→alight) × BUS_FACTOR (≈1.15, stops/dwell)
```
`road_delay_ratio` = TomTom time with traffic ÷ time without traffic for the same stop pair, cached per (stop pair, 15-min slot).
Waiting = headway/2 from how often the route runs, capped at 15 min. Bus options are always shown as a **range**.
Bus lanes (e.g. ORR) get a lower factor later, when we have the data to back it.

Quota note: this adds about 1 TomTom call per bus leg, but stop pairs repeat a lot between users, so the cache absorbs most of it.

### 3.4 Road providers: Strategy + Adapter
```python
class RoadRouter(Protocol):
    async def routes(self, o: LatLng, d: LatLng, mode: RoadMode,
                     depart_at: datetime | None, alternatives: int) -> list[RoadRoute]: ...
```
`TomTomRouter` (default; free plan about 2,500 non-tile requests/day, pricing changed 2026-07-01, so check again),
`GoogleRouter` (optional, paid, real `TWO_WHEELER` ETAs), `FakeRouter` (recorded fixtures, no network in tests).
Each adapter converts its provider's format into our own `RoadRoute`; core code never sees a provider's raw format.

### 3.5 Recommender (deterministic rules, no AI)
Every option has `duration_s`, `duration_range_s`, `cost_inr`, `walk_m`, `transfers`, `reliability`
(road at peak = low, metro = high, bus = low at peak).

```
fastest  = option with the lowest duration (upper end of its range)
if metro option within SAVE_MIN (5 min) of fastest and its reliability is higher → recommend metro
elif fastest is road and delay_ratio ≥ PEAK_RATIO (1.5)                         → recommend it + "leave at <advice>"
else                                                                             → recommend fastest
cheapest option is always labelled; "least walking" is labelled when it differs
```
Every option comes with short **reasons**. The thresholds live in config. User preferences ("I don't own a vehicle",
"max walk 1 km", "avoid autos") filter options **before** scoring.

### 3.6 Leave-time advisor
TomTom `departAt` (TomTom's predicted traffic) every 15 min across −60…+120 min around the requested time,
for the road option and the bus correction. Returns the earliest and latest departure where `delay_ratio < PEAK_RATIO`.
About 12 calls each, cached for 6 h. Once we have about 4 weeks of our own history, that replaces TomTom's prediction.

### 3.7 Quota guard + cache
**Serving the whole city on a free plan means cost control is part of the design.**
- Cache key: start and destination snapped to geohash-7 (~150 m), plus mode, plus a 5-min time bucket.
  TTL: 5 min live, 6 h predicted, 30 days geocoding, 15 min bus stop-pair ratios.
- Daily budget in `api_usage`, split by feature (plan 55%, bus correction 20%, watcher 15%, leave-time 10%).
  When a share runs out, that feature falls back to cached or history data and the UI says so ("traffic as of 9:05").
- Watch the cache hit rate from day one. It tells us when we have to move to a paid plan.

### 3.8 Commute watcher
Its **own process** (`python -m sari_daari.worker`, APScheduler), so several API workers don't each run the job.
Polls saved commutes every 15 min starting 60 min before the usual time, stores observations, and sends one deduplicated web push
when the road delay crosses the threshold or the recommendation changes.

### 3.9 Live traffic map
MapLibre GL with TomTom raster traffic-flow tiles over an OSM base map, fetched through
`/v1/tiles/traffic/{z}/{x}/{y}` (keeps the key out of the browser, 60 s cache). Metro lines are drawn from GTFS shapes, in the lines' official colours.

---

## 4. Data model (PostgreSQL 16)

```
devices(id, created_at, lang)                     -- anonymous; optional magic-link account later
places(id, device_id, label, lat, lon)            -- "Home", "Office"
preferences(device_id, vehicles text[], max_walk_m, avoid text[])
commutes(id, device_id, origin_place, dest_place, usual_depart time, days int[], active)
route_observations(id, commute_id null, o_cell, d_cell, mode, requested_at, depart_at,
                   duration_s, free_flow_s, distance_m, provider, alt_rank)
bus_leg_ratios(board_stop, alight_stop, slot, ratio, observed_at)   -- feeds the bus time correction
push_subscriptions(id, device_id, endpoint, p256dh, auth)
api_usage(provider, day, feature, count)
provider_cache(key, payload jsonb, expires_at)    -- UNLOGGED
```
Transit data (stops, routes, timetables) stays **in OTP and in memory**, not in Postgres. Feed updates
are file swaps, not migrations.

---

## 5. API (v1)

| Method | Path | Purpose |
|---|---|---|
| POST | `/v1/plan` | `{origin, destination, depart_at?, vehicles[], max_walk_m?, avoid[]?}` → options, recommendation, reasons |
| GET | `/v1/nearby?lat=&lon=` | nearest metro stations (entrance + gate) and bus stops, with walking time, routes served, how often they run |
| GET | `/v1/stops/{id}` | one stop: routes, directions, approximate next departures |
| GET | `/v1/leave-advice` | best departure window |
| CRUD | `/v1/commutes`, `/v1/places`, `/v1/preferences` | per device |
| GET | `/v1/geocode?q=` | proxied, cached search limited to the Bengaluru area |
| GET | `/v1/tiles/traffic/{z}/{x}/{y}` | proxied traffic tiles |
| POST | `/v1/push/subscribe` | web push |

Local time `Asia/Kolkata` at the edges, UTC in storage. Response text keys are ready for translation (en/kn).

---

## 6. Tech stack and hosting

| Tech | Role | Why |
|---|---|---|
| Python 3.13, FastAPI, Pydantic v2, httpx | API + planner | Your stack; the work is mostly waiting on network calls, which async handles fine |
| OpenTripPlanner 2 (Java 21+) | Transit routing | Proven RAPTOR + walking over streets; reads our feeds directly |
| SQLAlchemy 2 + Alembic, PostgreSQL 16 | State, history, cache, quota | Already installed natively (no Docker) |
| APScheduler (worker process) | Commute watcher | No Redis/Celery for now |
| scipy `cKDTree` | Nearby stops | Fast enough for 10k points |
| uv, ruff, pytest | Tooling | Fast and reproducible |
| Next.js + TypeScript, MapLibre GL, `openapi-typescript` | `web/` PWA | Mobile-first; TS types generated from FastAPI's OpenAPI spec |

**Hosting:** OTP's memory means **one always-on server with 4–8 GB RAM** (e.g. a Hetzner/DigitalOcean-class VPS,
roughly ₹500–1,500/month) running OTP + API + worker + Postgres. Frontend on Vercel. Free tiers that put the app to sleep
don't work: the watcher has to stay running, and OTP takes minutes to load its graph.

---

## 7. Repository layout

```
sari-daari/
├── backend/
│   ├── pyproject.toml
│   ├── src/sari_daari/
│   │   ├── api/            # FastAPI routers
│   │   ├── domain/         # LatLng, Stop, Leg, TripOption, Recommendation
│   │   ├── providers/      # RoadRouter: tomtom.py, google.py, fake.py
│   │   ├── transit/        # TransitRouter protocol, otp.py (GraphQL client), fake.py
│   │   ├── stops/          # gtfs loader, KD-tree index, nearby
│   │   ├── engine/         # plan.py, bus_correction.py, recommend.py, leave_time.py
│   │   ├── infra/          # db.py, cache.py, quota.py
│   │   ├── worker.py       # commute watcher
│   │   └── config.py
│   ├── migrations/
│   └── tests/              # fixtures: recorded TomTom + OTP responses
├── otp/                    # build-config.json, router-config.json, scripts to build the graph (graph itself gitignored)
├── web/                    # Next.js PWA
├── scripts/fetch_data.py   # GTFS zips + OSM extract → data/ (gitignored)
├── docs/
└── .env.example
```

---

## 8. Risks and how we handle them

| Risk | Handling |
|---|---|
| **BMTC timetables are wrong**, and only routes with working live tracking are in the feed | Traffic-corrected bus times shown as ranges (§3.3); "this route may be missing" note; collect user reports (v2) |
| Metro GTFS times are approximate | Show as ranges; fetch the feed weekly |
| No official live bus/metro data | **Don't** scrape the Namma BMTC app's private API. Ask BMTC/DULT for official access (they shared GTFS for the 2024 Enroute challenge) |
| TomTom quota with city-wide users | Cache + quota guard (§3.7); watch the hit rate; budget for a paid plan before public launch |
| OTP memory / ops | Single VPS sized for it; graph swap without downtime; `TransitRouter` interface keeps a lighter router possible |
| Bike ETAs are wrong (bikes filter through traffic) | v2 per-user correction from "how long did it take?" feedback |
| Getting an auto at a station is hard / surge pricing | Auto waiting time in config; per-station flag later |

## 9. Data licences

- **BMRCL GTFS** and **BMTC GTFS** (Vonter): **ODbL 1.0**. Credit "Vonter / BMRCL / BMTC" in the app. A changed copy of the database
  that we distribute stays ODbL. **Feeds are never committed**; `scripts/fetch_data.py` downloads them.
  The BMTC feed was built by a third party from the Namma BMTC app. Before any commercial launch, get official data from BMTC.
- **OpenStreetMap**: © OpenStreetMap contributors (ODbL).
- **TomTom**: follow their attribution rules on the map.
- OpenCity metro ridership (v2 crowding hint) is **CC BY-NC-SA**, so only while the app stays non-commercial.

---

## 10. Build order

1. **Phase 0: skeleton.** uv project, config, models, fake routers, ruff + pytest.
2. **Phase 1: data + OTP.** `fetch_data.py`, OSM cut to the city, OTP graph build, OTP client, a few known trips
   (Indiranagar → Majestic, Koramangala → Whitefield) as tests against recorded responses.
3. **Phase 2: nearby stops.** Loader + KD-tree + `/v1/nearby`, `/v1/stops/{id}`.
4. **Phase 3: TomTom + `/v1/plan`.** Road options, bus time correction, recommender, quota guard.
5. **Phase 4: web.** Plan screen, Nearby screen, map with traffic tiles, Kannada/English.
6. **Phase 5: saved commutes + watcher + leave-time + push.**
7. **Phase 6: rain, crowding, trip feedback, ask BMTC/BMRCL for official data.**
