# Sari-Daari — Architecture

> *Sari daari* (ಸರಿ ದಾರಿ): "the right way". For each trip it answers one question:
> **bike/car or metro, and when should I leave?**

Status: design (2026-10-16). Nothing below is implemented yet.

---

## 1. What we're building (and what we're not)

Google Maps already shows live traffic and alternate routes, so copying that is
not the product. The product is a **decision for each trip**:

> Your usual 9:00 ride to Whitefield: **62 min by bike today** (normally 35).
> **Metro door to door: 41 min**: 6 min walk, 8 min auto to Indiranagar, 22 min ride, 5 min walk.
> Or **leave by 8:20** and the bike takes 38 min.

| Feature | Version |
|---|---|
| Door-to-door comparison: bike / car / metro (+ walk or auto to and from the station) | first version |
| Live traffic on the chosen route + 2 alternate routes | first version |
| Peak-hour warning ("this is 1.8× normal, likely to get worse") | first version |
| Saved commutes (home ↔ office) | first version |
| Leave-time advice ("leave before 8:20 or after 10:40") | v1 |
| Push alerts before you usually leave | v1 |
| Rain adjustment, crowding on the metro | v2 |
| BMTC buses, our own predictions learned from history | v2+ |

Out of scope: turn-by-turn navigation (we hand off to Google Maps or the user's maps app),
ticketing (Namma Yatri and the BMRCL app already do it).

---

## 2. System overview

```mermaid
flowchart LR
  subgraph Client["web/ — Next.js PWA"]
    UI[Compare screen<br/>+ MapLibre map]
    SW[Service worker<br/>web push]
  end

  subgraph API["backend/ — FastAPI"]
    R[HTTP routes /v1]
    CMP[Comparison engine]
    REC[Recommender<br/>deterministic rules]
    LT[Leave-time advisor]
    TR[Metro router<br/>in-memory GTFS graph]
    QG[Quota guard + cache]
    W[Commute watcher<br/>APScheduler]
  end

  subgraph Providers["External"]
    TT[TomTom Routing<br/>+ Traffic tiles]
    GG[(Google Routes<br/>optional, two-wheeler)]
    OM[Open-Meteo<br/>rain, v2]
  end

  DB[(PostgreSQL 16)]
  GTFS[[BMRCL GTFS<br/>Vonter/bmrcl-gtfs]]

  UI -->|REST| R
  R --> CMP --> REC
  R --> LT
  CMP --> TR
  CMP --> QG --> TT
  QG -.-> GG
  LT --> QG
  W --> CMP
  W -->|delay crosses threshold| SW
  REC -.-> OM
  TR <-- loaded at startup --- GTFS
  CMP --> DB
  W --> DB
  UI -->|traffic tiles via proxy| R
```

Request flow for `POST /v1/compare`:

1. **Geocode** (if given text): TomTom Search, limited to the Bengaluru area.
2. **Road options**, run in parallel: TomTom `calculateRoute` with `traffic=true`,
   `maxAlternatives=2`, for `car` and `motorcycle`. We keep both
   `travelTimeInSeconds` and `noTrafficTravelTimeInSeconds`. Their ratio is the
   **delay ratio** behind every peak-hour warning.
3. **Metro options**: the metro router finds the k=3 nearest stations to the start and to the destination,
   prices each pair (walk or auto to the station + wait + ride + interchange + walk or auto at the other end),
   and keeps the best 1–2.
4. **Score and recommend** (deterministic, see §5).
5. Store an observation row (that's how history builds up), then return.

---

## 3. Components

### 3.1 Providers: Strategy + Adapter
```python
class RoadRouter(Protocol):
    async def routes(self, o: LatLng, d: LatLng, mode: RoadMode,
                     depart_at: datetime | None, alternatives: int) -> list[RoadRoute]: ...
```
- `TomTomRouter`: the default. Free plan: about **2,500 non-tile requests/day** (commercial use OK);
  traffic tiles have a separate ~50k/day quota. *Check the numbers again: TomTom's pricing changed 2026-07-01.*
- `GoogleRouter`: optional, for real `TWO_WHEELER` ETAs in India. Paid once traffic data is used.
- `FakeRouter`: recorded fixtures for tests. Tests never make network calls.

Each adapter converts its provider's format into our own `RoadRoute`
(`duration_s`, `free_flow_s`, `distance_m`, `polyline`, `incidents[]`, `provider`).
The core code only ever uses our models, never a provider's raw format.

### 3.2 Metro router (in-process, no OpenTripPlanner)
Namma Metro is small (3 lines, about 80 stations), so a JVM OpenTripPlanner server would be overkill.
We load the **frequency-based** feed (`bmrcl-frequencies.zip`) at startup:

- `stops` (location_type=1 → stations, with platforms and entrances underneath them),
  `routes`/`trips`/`stop_times` → time between neighbouring stations, `frequencies` → how often trains run by time of day,
  `transfers` (`min_transfer_time`) → interchange penalty (Majestic, RV Road, …),
  `fare_rules`/`fare_attributes` → fare between two stations.
- Graph: one node per station and line, one edge per segment, plus interchange edges. Search with Dijkstra
  over a graph that takes a few KB.
- Waiting for a train = `headway/2` for the time slot. The feed's `exact_times=1` timings are
  approximate (the author warns times can be off by a couple of minutes), so we show the metro ride as **a range**, not a single number.
- Walking or an auto to and from the station: walk if ≤ 1.2 km (fixed 80 m/min × 1.3 detour factor in the first version,
  OSRM foot routing later), otherwise auto = TomTom car ETA × 1.0 + 5 min
  waiting for an auto. Both values live in config and can be tuned.

Use station **entrances** (`location_type=2`) when measuring walking distance. Big stations
have entrances hundreds of metres apart.

### 3.3 Comparison engine
Builds `TripOption`s, each a list of `Leg`s (`walk | auto | metro | bike | car`), and fills in:
- `duration_s` (plus `duration_range_s` for anything with metro or prediction in it)
- `cost_inr`: fuel = km × ₹/km per vehicle (config), + parking (config, per destination later),
  metro fare from GTFS, auto from a config table of BLR meter rates
- `delay_ratio` for road options

### 3.4 Recommender (deterministic rules, no AI)
```
road_best  = fastest of bike/car the user owns
metro_best = fastest metro option
if metro_best.duration < road_best.duration - SAVE_MIN (5 min)       → "Take metro"
elif road_best.delay_ratio ≥ PEAK_RATIO (1.5) and metro within 10 min → "Metro is about as fast and more predictable"
elif road_best.delay_ratio ≥ PEAK_RATIO                              → "Peak traffic. Consider leaving at <advice>"
else                                                                  → "Road is fine"
```
Every recommendation comes with **reasons** (a list of short strings) so the UI can say *why*.
The thresholds are in config. Rules first, because they're easy to explain and test; a learned model can come later.

### 3.5 Leave-time advisor
Calls TomTom with `departAt` (TomTom's predicted traffic) every 15 min
across a window (default −60 … +120 min around the requested time), then finds the earliest and latest
departures where `delay_ratio < PEAK_RATIO`. One piece of advice costs about 12 calls, so it's cached
hard (see 3.6). Once we have about 4 weeks of our own history (§3.7), that replaces the prediction.

### 3.6 Quota guard + cache
The free plan means **cost control is part of the design**, not something to bolt on later.
- Cache key: start and destination snapped to a ~150 m grid (geohash-7), plus mode, plus a 5-min time bucket.
  TTL: 5 min for live traffic, 6 h for `departAt` predictions, 30 days for geocoding.
- Daily budget in `api_usage` (provider, day, count), split by feature:
  live compare 60%, watcher 25%, leave-time 15%. When a share runs out, that feature
  falls back to cached or history data and the UI shows it (e.g. *"traffic as of 9:05"*).
- Cache lives in Postgres (`UNLOGGED` table) to start. Redis only if we ever need it.

### 3.7 Commute watcher (background)
APScheduler running in the same process as the API (one worker in the first version, no queue).
For each saved commute on a commute day, poll every 15 min starting 60 min before the usual time.
Each poll writes a `route_observations` row. If the delay ratio crosses the threshold, or metro
becomes the better choice, it sends one web push to that commute (deduplicated per departure window).

Rough budget: 50 users × 2 commutes × 5 polls ≈ **500 calls/day**. That fits.

### 3.8 Live traffic map
Frontend: MapLibre GL with TomTom's raster **traffic flow tiles** on an OSM/TomTom base map.
Tiles are fetched through a backend proxy `/v1/tiles/traffic/{z}/{x}/{y}` so the API key never reaches the browser,
and the tiles get a short cache (60 s).

---

## 4. Data model (PostgreSQL 16)

```
users(id, device_id, created_at)                  -- anonymous device id first; magic-link auth later
places(id, user_id, label, lat, lon)              -- "Home", "Office"
commutes(id, user_id, origin_place, dest_place, usual_depart time, days int[],
         vehicles text[], active bool)
route_observations(id, commute_id null, o_cell, d_cell, mode, requested_at,
         depart_at, duration_s, free_flow_s, distance_m, provider, alt_rank)
         -- the history everything else learns from; partition by month once it gets big
actual_trips(id, user_id, commute_id, mode, started_at, ended_at)   -- "how long did it take?" feedback (v1)
push_subscriptions(id, user_id, endpoint, p256dh, auth)
api_usage(provider, day, feature, count)          -- quota guard
provider_cache(key, payload jsonb, expires_at)    -- UNLOGGED
```
Metro data stays **in memory** and is loaded from GTFS at startup, not stored in the database. It's static
and small, and putting it in the database would turn every feed update into a migration.

`actual_trips` matters: bikes filter through Bengaluru traffic, so no provider's "motorcycle" ETA
is exact. Once we have real trip times, a correction factor per user and route makes the bike ETA honest.

---

## 5. API (v1)

| Method | Path | Purpose |
|---|---|---|
| POST | `/v1/compare` | `{origin, destination, depart_at?, vehicles[]}` → options + recommendation + reasons |
| GET | `/v1/leave-advice` | best departure window for an origin/destination/mode |
| GET/POST/PATCH/DELETE | `/v1/commutes` | saved commutes |
| POST | `/v1/trips` | report how long a trip actually took |
| GET | `/v1/metro/stations` | stations + lines (for the map) |
| GET | `/v1/geocode?q=` | proxied, cached TomTom search limited to the Bengaluru area |
| GET | `/v1/tiles/traffic/{z}/{x}/{y}` | proxied traffic tiles |
| POST | `/v1/push/subscribe` | register for web push |

Times are local time `Asia/Kolkata` at the edges and UTC in storage.

---

## 6. Tech stack

| Tech | Role | Why |
|---|---|---|
| Python 3.13, FastAPI, Pydantic v2 | API + domain models | Same stack as PayWeave/Sentinel; typed request and response contracts |
| httpx (async) | Provider calls | Timeouts, retries, easy to mock |
| SQLAlchemy 2 + Alembic | Database access + migrations | Standard, async |
| PostgreSQL 16 | State, history, cache, quota | Already installed natively on the dev Mac (no Docker) |
| APScheduler | Commute watcher | Runs in-process; no Redis/Celery in the first version |
| `gtfs-kit` or plain `csv` | Load GTFS | Small feed; plain CSV probably wins |
| uv, ruff, pytest | Tooling | Fast and reproducible |
| Next.js (App Router) + TypeScript | `web/` PWA | Mobile-first; built-in PWA/web-push support |
| MapLibre GL JS | Map | Open source, no Mapbox token |
| Open-Meteo | Rain (v2) | Free, no API key |

Deploy (later): API on Fly.io or Render free tier with Neon Postgres. Web on Vercel.

---

## 7. Repository layout

```
sari-daari/
├── backend/
│   ├── pyproject.toml
│   ├── src/sari_daari/
│   │   ├── api/            # FastAPI routers
│   │   ├── domain/         # models: LatLng, Leg, TripOption, Recommendation
│   │   ├── providers/      # RoadRouter protocol, tomtom.py, google.py, fake.py
│   │   ├── metro/          # gtfs_loader.py, graph.py, router.py
│   │   ├── engine/         # compare.py, recommend.py, leave_time.py
│   │   ├── infra/          # db.py, cache.py, quota.py, scheduler.py
│   │   └── config.py
│   ├── migrations/
│   └── tests/              # fixtures/ = recorded TomTom responses
├── web/                    # Next.js PWA
├── scripts/fetch_gtfs.py   # downloads the BMRCL feed into data/ (gitignored)
├── docs/
└── .env.example
```

---

## 8. Risks and how we handle them

| Risk | Handling |
|---|---|
| Metro GTFS is unofficial and only approximately right | Show metro time as a range; fetch the feed weekly; credit it |
| No live metro data (delays, breakdowns) | v2: watch BMRCL's official X/Twitter and news for disruptions; manual "disruption" flag |
| Bike ETAs are wrong (bikes filter through traffic) | Correction factor per user from `actual_trips` |
| TomTom quota | Quota guard + cache (§3.6); Google kept as a paid fallback |
| Auto availability/surge at stations | Waiting for an auto is in config; v2 adds a per-station "hard to get an auto" flag |
| Peak crowding on the metro | v2: OpenCity hourly ridership as a crowd hint. **The licence is CC BY-NC-SA, so only while non-commercial** |

## 9. Data licences

- **BMRCL GTFS** (Vonter/bmrcl-gtfs): **ODbL 1.0**. Credit "Vonter / BMRCL" in the app. A changed
  copy of the database that we distribute must also be ODbL. We **don't commit the feed**; `scripts/fetch_gtfs.py` downloads it.
- **TomTom**: follow their attribution rules on the map.
- **OpenStreetMap**: © OpenStreetMap contributors (ODbL).

---

## 10. Build order

1. **Phase 0: skeleton.** uv project, config, models, FakeRouter, CI (ruff + pytest).
2. **Phase 1: metro router.** GTFS loader + graph + fares, tested on known journeys
   (e.g. Indiranagar → Majestic, Whitefield → Silk Institute with the interchange).
3. **Phase 2: TomTom adapter + `/v1/compare`.** Recorded fixtures, delay ratio, recommender rules.
4. **Phase 3: web.** Compare screen, map with route lines and traffic tiles.
5. **Phase 4: saved commutes + watcher + history.**
6. **Phase 5: leave-time advice + web push.**
7. **Phase 6: rain, crowding, trip feedback correction, buses.**
