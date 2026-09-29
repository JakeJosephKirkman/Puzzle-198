# Puzzle 198: Smart Delivery Route Planner. Working plan

Status: plan only. No application code yet. Open questions are at the end; the
build does not start until they are answered.

Working name: **Durbacti Runsheet** (alternatives in section 9).

---

## 1. What we are building

A single-page web app where a dispatcher enters a depot and a list of
deliveries, and the app works out an efficient order to visit them. It must
beat the naive approach in two ways the puzzle clue calls out:

1. Not just visiting stops in the order they were typed in.
2. Not just "go to the nearest unvisited stop" (nearest neighbour), which looks
   clever and is often 20 to 30 percent worse than a properly searched route.

Every requirement from the brief maps to a feature below. The mapping table in
section 8 is the checklist we tick off before calling it done.

## 2. Proposed stack (defaults, confirm in Q1 to Q3)

| Layer | Choice | Why |
|---|---|---|
| App | Vite + React + TypeScript | Fast to build, typed optimiser code, easy to unit test |
| Map | Leaflet + OpenStreetMap tiles, Esri World Imagery for the satellite toggle | No API key, no billing, works on GitHub Pages |
| Geocoding | Nominatim (OpenStreetMap) with debounce, plus click-on-map | Free, good enough for a demo, one request per second limit |
| Distances | Haversine matrix by default, optional OSRM table service for road distances and road geometry | Haversine is instant and offline. OSRM gives real road km and draws the route along streets |
| Optimiser | Pure TypeScript, runs in a Web Worker | Own algorithm as the brief requires, and the UI never freezes |
| Drag and drop | dnd-kit (or native HTML5 drag) | Row reorder in the deliveries table |
| State | Zustand (or plain React context) with localStorage persistence and JSON import/export | Survives refresh, shareable |
| Tests | Vitest | Optimiser correctness and a benchmark harness |
| Styling | CSS modules or Tailwind, dark theme first, light theme second | Matches the mockup |
| Hosting | GitHub Pages via Actions | Zero cost, public demo link |

Australian spelling throughout the UI (optimise, colour, kilometre). Units km
and minutes.

## 3. Data model

```ts
type Priority = 'high' | 'normal' | 'low';

interface Depot { id: 'depot'; label: string; lat: number; lng: number; startTime: string /* "08:00" */; }

interface Delivery {
  id: string;
  address: string;
  lat: number; lng: number;
  packages: number;            // or weight in kg, see Q7
  priority: Priority;
  window?: { start: string; end: string };  // "09:00" to "11:00", optional
  windowHard?: boolean;         // hard = infeasible if late, soft = penalised
  serviceMinutes?: number;      // override, else derived from packages
  locked?: boolean;             // optimiser may not move this stop
  notes?: string;
  status: 'scheduled' | 'unscheduled';   // unscheduled = did not fit capacity
}

interface Vehicle {
  id: string; label: string;
  capacity: number;             // packages or kg
  avgSpeedKmh: number;          // used when OSRM durations are unavailable
  shiftEnd?: string;            // driver must be back by
}

interface Settings {
  distanceMode: 'haversine' | 'road';
  roadFactor: number;           // haversine multiplier when road mode is off, ~1.3 for cities
  baseServiceMinutes: number;   // per stop
  perPackageMinutes: number;
  weights: { distance: number; late: number; early: number; priority: number; };
  allowWaiting: boolean;        // arrive early and wait, or penalise
  allowDepotReload: boolean;    // multi-trip when capacity is exceeded
  optimiserBudgetMs: number;    // search time budget
  seed?: number;                // deterministic runs for testing
}

interface RouteResult {
  order: string[];              // delivery ids in visit order (trips separated by 'depot' if reloads enabled)
  legs: Leg[];                  // per hop: from, to, km, minutes
  stops: StopSchedule[];        // arrival, wait, departure, lateMinutes, loadAfter
  totalKm: number; totalMinutes: number;
  cost: CostBreakdown;          // distance, lateness, priority, capacity terms
  feasible: boolean;            // false if a hard window or capacity is violated
}
```

Two routes are always held in state: `original` (the order the user entered or
dragged into) and `optimised` (the optimiser's answer). The comparison panel
diffs them.

## 4. Distance layer

- Build an N+1 by N+1 matrix (depot plus every delivery) of kilometres and
  minutes. Recompute only the rows and columns that change when a stop is
  added, moved or removed.
- Haversine mode: great-circle distance times `roadFactor`, minutes from
  `avgSpeedKmh`. Instant, no network.
- Road mode: one OSRM `table` request gives every pair's road distance and
  duration. Cache by coordinate set. Fall back to haversine if the request
  fails, and show a small "straight-line estimate" badge so the user knows.
- Route geometry on the map: straight polylines in haversine mode, OSRM
  `route` geometry in road mode.

## 5. Schedule simulation (the thing the cost function scores)

Given an order, walk it from the depot:

```
time = depot.startTime; load = sum(packages)
for each stop:
  time += travelMinutes(prev, stop)
  arrival = time
  if arrival < window.start and allowWaiting: wait = window.start - arrival
  lateMinutes = max(0, arrival + wait - window.end)
  departure = arrival + wait + serviceMinutes(stop)
  load -= stop.packages
  record {arrival, wait, departure, lateMinutes, loadAfter}
time += travelMinutes(last, depot)
```

This produces the itinerary table directly (arrival and departure per stop) and
feeds the cost function.

## 6. The optimiser

### 6.1 Cost function (what "efficient" means here)

```
cost(order) =
    w.distance * totalKm
  + w.late     * sum( lateMinutes_i * priorityMultiplier_i )        // high 3x, normal 1x, low 0.5x
  + w.early    * sum( waitMinutes_i )                                // small, waiting is wasted time
  + w.priority * sum( priorityRank_i * positionFraction_i )          // nudges high priority earlier
  + BIG        * hardViolations                                      // hard windows, shift end
```

Capacity is not a penalty term. It is a hard constraint handled structurally
(section 6.4). Locked stops are also structural (6.5).

Every term is reported separately in a cost breakdown panel so the user can
see why the optimiser chose what it did, and can turn the weight sliders to
change the trade-off and re-run.

### 6.2 Construction (starting points)

The search starts from several candidate orders and keeps the best:

1. The user's original order.
2. Nearest neighbour (kept as the "naive" baseline so the comparison panel can
   show how much better the real search does).
3. Cheapest insertion: start with the depot, repeatedly insert the stop whose
   best insertion position adds the least cost. Time-window aware because it
   scores with the full cost function, not just distance.
4. Two or three randomised insertions (seeded) for diversity.

### 6.3 Improvement (local search)

Applied to each start until no move improves the cost:

- **2-opt**: reverse a segment. Removes crossing legs.
- **Or-opt**: relocate a segment of 1 to 3 stops elsewhere. Good for time
  windows because it moves a stop earlier or later without disturbing the rest.
- **Swap**: exchange two stops.

Then a metaheuristic on top so we do not get stuck in the first local optimum:

- **Simulated annealing** or **iterated local search** with a double-bridge
  perturbation, run against `optimiserBudgetMs`, reporting best-so-far to the
  UI every ~100 ms so the KPI tiles visibly tick down while it searches.

For instances with 10 or fewer free stops, an exact Held-Karp dynamic programme
runs as well and the UI can show "proven optimal for distance" when it matches.
That also gives the test suite a ground truth.

### 6.4 Capacity

- Total packages greater than vehicle capacity is the interesting case.
- Default behaviour: the optimiser keeps the highest-value stops (priority first,
  then earliest window) up to capacity and marks the rest **Unscheduled**. They
  stay in the list with a badge and are not routed.
- Optional (`allowDepotReload`): the route may return to the depot to reload,
  which turns the problem into multi-trip. Reload legs cost distance and time
  like any other leg, so the search only uses them when worth it.
- The load column in the itinerary shows load after each stop, and the capacity
  bar in the header turns red if a manual drag produces an overload.

### 6.5 Locked stops

A locked stop keeps its position index. Every operator works on the unlocked
slots only: a 2-opt reversal over slots i to j takes the unlocked elements in
that range, reverses them, and drops them back into the unlocked slots, leaving
the locked ones exactly where they were. Or-opt and swap check the lock flag
and skip. This is simpler and faster than generating moves and rejecting them.

### 6.6 Why nearest neighbour loses (for the write-up and the demo)

Nearest neighbour commits to the closest stop every time and ends up with a few
very long legs at the end, plus it ignores time windows entirely. The demo will
include a "show me" button that runs nearest neighbour and the full search on
the same data and animates both routes so the difference is visible.

## 7. UI plan (matches the mockup layout)

- **Header**: KPI tiles (deliveries, total km, estimated time, time saved, %
  shorter), Optimise button, Compare Routes toggle, theme toggle, vehicle picker.
- **Sidebar**: All / Unscheduled / Scheduled / Vehicles / Settings.
- **Map**: markers numbered in visit order, green depot start, chequered finish,
  optimised route in blue, original route dashed grey when Compare is on,
  legend, map/satellite toggle, locate me.
- **Deliveries panel**: table with # / address / window / packages / priority /
  lock. Rows drag to reorder. Dropping a row recalculates the schedule
  immediately and marks the route as "manual" until Optimise is pressed again.
  Search box, Add Delivery form (address search or click on map), Edit All.
- **Route Comparison panel**: original vs optimised distance and time, bars,
  distance and time saved.
- **Itinerary panel**: per stop arrival, wait, departure, late flag, load after.
  Export as CSV and print view.
- **Cost breakdown drawer**: the terms from 6.1 for the current route, with
  weight sliders.
- **Algorithm lab (stretch)**: pick construction and improvement steps, see a
  results table and a convergence chart.

## 8. Requirement checklist

| Brief requirement | Where it lands |
|---|---|
| Interactive map | Leaflet, section 7 |
| Add multiple delivery locations | Add Delivery form, geocode or map click |
| Every delivery as a marker | Numbered markers |
| Reorder manually | Drag rows |
| Distance between every location | Distance matrix, section 4 |
| Own optimisation algorithm | Section 6, no external solver |
| Efficient visiting order | Construction + local search + annealing |
| Route on map | Polylines (straight or road geometry) |
| Total distance, estimated time | KPI tiles |
| Numbered stops | Markers and table |
| Priority per stop | Priority field, cost multiplier |
| Time windows | Window field, schedule simulation |
| Penalise arriving outside window | Late and early terms in cost |
| Max vehicle capacity | Vehicle settings |
| Weight or packages per delivery | Packages field |
| Never exceed capacity | Unscheduled overflow or depot reload |
| Lock stops | Lock flag, section 6.5 |
| Original vs optimised | Two routes in state, compare panel |
| Distance and time saved | Comparison panel and KPI |
| Drag to reposition and recalc | Drag handler triggers schedule rebuild |
| Clear itinerary | Itinerary panel |

## 9. Naming (relating to Durbacti)

Durbacti is a Sydney AI automation agency, so the name should read as a
product from that shop, be sayable, and not be cute for its own sake.

| Name | Read | Notes |
|---|---|---|
| **Durbacti Runsheet** (recommended) | "run sheet" is what Australian drivers and tradies call the day's list of jobs in order | Exactly what the app produces. Local, plain, memorable |
| Durbacti Dispatch | The desk that decides who goes where | Generic but clear |
| Durbacti Drops | Deliveries are "drops" | Short, friendly |
| Durbacti Milk Run | Logistics term for a fixed multi-stop loop | Charming, slightly obscure |
| Durbacti Waypoint | Navigation flavour | Fine, a bit corporate |
| DurbacTactix | Pun on the "acti" in Durbacti and "tactics" | Too cute for the house style |

Tagline candidates for the sidebar card: "Fewer kilometres. Same drops." or
"Plan the run. Keep the windows."

Default map centre: Sydney CBD, with the demo dataset spread across Parramatta,
Northern Beaches and the Sutherland Shire (Durbacti's stated service area).

## 10. Additional feature ideas (beyond the brief)

Grouped by how much they add to the optimiser versus the UI.

**Optimiser depth**
- Multi-vehicle assignment (split stops across 2 or more vans, each with its own
  capacity and shift). This is the real jump in complexity: a capacitated VRP.
- Driver breaks (a 30 minute break must fall between hour 4 and 5).
- Avoid zones: draw a polygon (road closure, flood, event), and any leg crossing
  it is penalised or forbidden. The mockup already shows one.
- Time-of-day speed profiles (peak hour slows legs that fall in 07:30 to 09:30).
- Pickup and delivery pairs (collect at A, must drop at B later in the route).
- Exact solver for small instances to prove the gap (Held-Karp, section 6.3).
- Benchmark mode: 100 random instances, report nearest neighbour versus full
  search, mean and worst-case improvement. Good content for a write-up.

**Operational realism**
- Live simulation: play the route with a slider, van marker moves, ETA per
  stop updates. "Driver is 20 minutes late at stop 4, re-plan the rest."
- Mark a stop delivered or failed mid-route and re-optimise the remainder from
  the current position.
- Multi-day: stops that cannot fit today roll to tomorrow's run.
- Fuel and CO2 estimate from km and vehicle type.
- Customer notifications preview (SMS text with the ETA window).

**Data in and out**
- CSV import of addresses with a mapping step, bulk geocode with progress.
- JSON save and load, and a share link that encodes the whole plan in the URL.
- Export itinerary as CSV, PDF print view, and GPX for a sat nav.
- Demo datasets: Sydney 12 stops, Sydney 40 stops, a deliberately nasty one
  with tight windows and an overload.

**Product polish**
- Undo and redo for reorders and edits.
- Keyboard shortcuts (O optimise, C compare, L lock selected).
- Light and dark themes, colour-blind safe priority badges.
- Hover a table row to highlight its marker and legs, and vice versa.
- Accessible table (drag has a keyboard fallback: move up and move down).

## 11. Testing plan

- Unit: haversine against known city pairs, schedule simulation with waits and
  lateness, cost function terms, each operator respects locks, capacity
  overflow produces correct unscheduled set.
- Property: for random instances up to 9 stops, the search never returns worse
  than nearest neighbour and matches Held-Karp on distance-only cost.
- Snapshot: the demo dataset's itinerary.
- UI smoke: Playwright, add three stops, drag one, optimise, compare, export.

## 12. Milestones

1. Scaffold, map, add and geocode stops, markers, table.
2. Distance matrix, schedule simulation, itinerary, KPI tiles.
3. Optimiser v1 (construction + 2-opt + Or-opt) in a worker, compare panel.
4. Constraints: priority, windows, capacity with unscheduled overflow, locks.
5. Drag reorder with recalculation, road mode via OSRM, export, persistence.
6. Tests and benchmark harness, GitHub Pages deploy.
7. Stretch features from section 10, chosen from the answers below.

## 13. Open questions

Each has a suggested default so a one-word answer is enough.

1. **Stack.** Vite + React + TypeScript, or a single HTML file with no build
   step? Default: Vite + React + TS.
2. **Map provider.** Leaflet with OpenStreetMap (no key), or Mapbox or Google
   (key needed, nicer satellite)? Default: Leaflet + OSM.
3. **Distances.** Straight-line only, or real road distances and street-hugging
   route lines via the public OSRM server (rate-limited demo server, may be
   slow)? Default: both, with straight-line as the offline fallback.
4. **Capacity overflow.** When stops exceed the van: leave the lowest-value
   stops unscheduled, allow depot reloads (multi-trip), or split across
   multiple vehicles? Default: unscheduled, with reload as a toggle.
5. **Multiple vehicles.** In scope now (real VRP, the mockup shows 2 vehicles),
   or single vehicle first and multi-vehicle as milestone 7? Default: single
   first.
6. **Time windows.** Soft (penalised) everywhere, hard everywhere, or a per-stop
   toggle? Default: per-stop toggle, soft by default.
7. **Capacity unit.** Package count, weight in kg, or both? Default: packages,
   with an optional kg field.
8. **Priority meaning.** High priority means (a) visited earlier, (b) lateness
   hurts more, or (c) both? Default: both.
9. **Depot.** Start and finish at the same depot, or allow a different end point
   (driver goes home)? Default: same depot, end point optional.
10. **Region.** Sydney demo data (Durbacti's home) or London as in the mockup?
    Default: Sydney.
11. **Persistence.** localStorage plus JSON export only, or a Supabase backend
    so plans are saved per user? Default: localStorage, Supabase as stretch.
12. **Live simulation.** Include the van playback and "driver running late,
    re-plan" feature, or leave it out? Default: include as milestone 7.
13. **Avoid zones and traffic profiles.** Worth building? Both add real
    optimiser complexity. Default: avoid zones yes, traffic profiles later.
14. **Algorithm lab.** Do you want the exposed controls (choose construction
    and improvement, convergence chart, benchmark mode), or keep the optimiser
    a black box behind one button? Default: lab included, hidden behind an
    "Advanced" toggle.
15. **Deployment.** GitHub Pages public demo, or local only? Default: Pages.
16. **Name.** Durbacti Runsheet, or another from section 9?
