# Rajkot GTFS Graph Engine & Algorithmic Pathfinder

**[Live Demo](https://LabibaMenal.github.io/rajkot-transit-analyser)**

An interactive dashboard that stress-tests Rajkot's real city bus network - click a road to see what breaks if it closes, check how much of the city is reachable from any stop, and find out which parts of the network have zero backup if something goes wrong. Runs entirely in the browser against Rajkot's real GTFS feed (716 stops, 903 shared road segments, 168 routes), with BFS and Dijkstra doing the actual graph work client-side.

---

## Why I Built This

This didn't start as a dashboard idea. The original assignment from my professor was something else: take Rajkot's GTFS feed and produce a schematic transit map using **LOOM** (Line-Ordering Optimization for Maps), a research tool out of the University of Freiburg - reproduce for Rajkot what their Stuttgart demo does. LOOM is C++ and only builds cleanly on Linux, so that meant setting up WSL, compiling it from source, and running the actual `gtfs2graph → loom → transitmap` pipeline on the raw GTFS folder (full breakdown below).

That first static map was genuinely useful even before I wrote a single line of dashboard code- it showed a central corridor where 6–8 routes were all crammed onto the same road, and one long route running south with no parallel backup at all. But an SVG is dead. You can look at it, you can't click a road and ask "okay, what would happen to the network if this closes." So instead of stopping at the image, I took the same underlying idea LOOM uses internally- stops as nodes, shared road segments as weighted edges- and rebuilt it myself as a GeoJSON graph, loaded it into Leaflet.js, and wrote BFS and Dijkstra on top of it (algorithms that i was learning at that time and felt it could be applied here for optimisation purpose) so the network could actually be interrogated instead of just looked at. That rebuild is this repo.

---

## What It Does

| Feature | DSA Concept | What You See |
|---|---|---|
| 🔥 Vulnerability Heatmap | Edge weight / centrality | Roads colored green→red by how many routes share them |
| 🚧 Road Block Simulator | BFS / connected components | Click a road to block it - watch stops get isolated live |
| 📡 Reachability Explorer | BFS with hop limit | Click a stop, see how much of the city you can reach |
| ⚠️ Dead Zone Finder | Graph degree analysis | 39 stops with zero backup route, highlighted in amber |
| 🧭 Shortest Path | Dijkstra's algorithm | Click 2 stops, see the least-distance route drawn on the map |

---

## Background: The LOOM/GTFS Exploration This Originated From

This section documents the exploratory phase- it's not part of the live dashboard, but it's where the graph concept and the key findings actually came from, and it's arguably the more technically involved half of the project.

**1. Linux environment.** LOOM's build system and dependencies only work properly on Linux, so on Windows that means WSL first. I installed Ubuntu-on-WSL as the base.

**2. Build dependencies.**
```bash
sudo apt-get update
sudo apt-get install build-essential cmake libboost-all-dev zlib1g-dev
```

**3. Clone and compile LOOM from source** (no prebuilt binary exists for this):
```bash
git clone https://github.com/ad-freiburg/loom
cd loom
mkdir build && cd build
cmake ..
make -j4
```

**4. Run the actual pipeline** - three LOOM tools chained together on the Rajkot GTFS feed:
```bash
unzip rrl_GTFS_wo_shapes_v15_Apr25.zip -d rajkot_gtfs

./build/gtfs2graph rajkot_gtfs > rajkot_graph.json
cat rajkot_graph.json | ./build/loom > rajkot_loom.json
cat rajkot_loom.json | ./build/transitmap > rajkot_map.svg
```
`gtfs2graph` turns the GTFS folder into a graph - stops become nodes, and any road segment shared by consecutive stops across trips becomes an edge. `loom` solves an ordering problem so overlapping routes on the same road don't visually collide. `transitmap` renders the result as a schematic SVG, in the same style as LOOM's Stuttgart example.

**What the map revealed:** a heavily overlapping central corridor carrying 6–8 routes on the same road, and one long, completely isolated route running south with zero redundancy - exactly the two patterns that later became the Vulnerability Heatmap and Dead Zone Finder features on the dashboard. In real transit-planning terms, the solution for the first problem is route rationalization (merge overlapping central routes, free up buses for underserved areas); the fix for the second is adding a parallel route so one failure doesn't kill an entire zone's service.

---

## The GTFS Data - What Actually Went In

The feed is `rrl_GTFS_wo_shapes_v15_Apr25` - RRL is Rajkot Rajpath Limited, the official operator. `v15` / `Apr25` means it's the 15th revision, last updated April 2025. `wo_shapes` matters: there's no `shapes.txt`, so there's no real road-curve geometry - only straight lines between consecutive stops. The whole feed is ~2.95 MB across 7 files: 168 routes, 716 stops, 2,733 trips, and 82,603 stop-time records.

Only a subset of columns across those files were actually used for the map generation:

| File | Columns used | Why | Ignored |
|---|---|---|---|
| `stops.txt` | `stop_id`, `stop_lat`/`stop_lon`, `stop_name` | Coordinates place every stop, name labels it | `stop_code`, `stop_desc` |
| `stop_times.txt` (82,603 rows) | `trip_id`, `stop_id`, `stop_sequence` | `stop_sequence` is what draws every edge - consecutive stops in a trip become a line segment | `arrival_time`, `departure_time` (real, but not needed to draw the network - only for a future delay-tracking feature) |
| `trips.txt` | `route_id`, `trip_id` | Links a sequence of stops back to the route it belongs to | `service_id` (only one service exists, so no filtering effect), `shape_id` (no `shapes.txt` to point to) |
| `routes.txt` | `route_id`, `route_short_name` | Groups and labels routes | `route_long_name`, `route_desc`. There's also no `route_color` column in this feed, so LOOM auto-assigned colors - the colors on the SVG aren't official RRL branding |
| `calendar.txt` | - | One service ID, all 7 days, Jan 2025 → Jan 2040 - so every trip is just always active | entire file, effectively |
| `agency.txt`, `feed_info.txt` | - | Operator/metadata only, no structural role | entire files |

This is also static scheduled data, not real-time GPS - it's the timetable RRL publishes, not live bus positions. That distinction is why "real-time delay tracking" shows up as a future idea below rather than something the current graph can do.

---

## How It Works (Architecture)

```
data/rajkot_graph.geojson  (716 point features, 903 LineString features)
            │
            ▼
        init()  →  parses points + lines, computes edgeWeights[] once
            │       (Haversine distance summed over each edge's coordinates)
            │
   ┌────────┼──────────┬───────────────┬───────────────┐
   ▼        ▼          ▼               ▼               ▼
Heatmap   Block Sim   Reachability   Dead Zones      Shortest Path
(static   (BFS from   (BFS capped    (precomputed:   (Dijkstra using
route     every stop  at N hops      degree === 1)   Haversine-km
count     after each  from clicked                   edge weights)
per       block)      stop)
edge)
```

The GeoJSON itself just stores raw stops and edges (with the routes riding each edge). Everything else - route counts per edge, connected components, hop-limited reachability, degree, and edge distance in km - is derived once at load time or on each click, not pre-baked into the file.

- **BFS features** (block simulator, reachability, dead zones) treat every edge as 1 hop, complexity O(V + E) = O(716 + 903) per query - near instant.
- **Dijkstra** (shortest path) is a plain O(V²) min-scan with no heap - at 716 nodes that's ~512K ops worst case, still instant for a single click, and simpler to reason about than a heap-based version at this graph size.

---
## The Dashboard

### Data

The underlying data file is `data/rajkot_graph.geojson` - generated by the `gtfs2graph` step above. It encodes:

- **716 Point features** (bus stops): each has an ID, stop name, GPS coordinates, and degree (number of connected edges)
- **903 LineString features** (road edges): each has a `from` and `to` stop ID, plus a `lines` array listing every route that uses that edge. The most loaded single edge has 79 routes on it simultaneously.

All graph analysis in the dashboard (BFS, Dijkstra, degree analysis) runs directly on this GeoJSON at page load - no backend, no database.

### Algorithms Used

**BFS (Breadth-First Search)** is used in two features. The Road Block Simulator uses it to find connected components after edges are removed - any stop not reachable from the main component gets flagged as isolated. The Reachability Explorer uses it to expand outward from a selected stop up to N hops, where each hop represents one bus segment (roughly 3 minutes of travel).

**Dijkstra's Algorithm** (shortest path, Feature 5) builds a weighted adjacency list at runtime where each edge weight is the real-world distance in km, computed via the Haversine formula applied to the edge's coordinate list. It then runs a standard O(V²) min-scan Dijkstra - 716 nodes means at most ~512K comparisons, which completes in under a millisecond in the browser.

**Degree Analysis** (Dead Zone Finder) is the simplest feature algorithmically - it just filters stops by `deg === 1`, but it recognizes the most actionable insight: 39 stops (5.4% of the network) have exactly one route and zero redundancy.

---

## Tech Stack

- HTML + vanilla JavaScript - no frameworks, single `index.html`
- Leaflet.js v1.9.4 for map rendering
- CartoDB Dark tiles as the basemap
- GitHub Pages for deployment
- *(Exploratory phase only, not part of the running app)*: LOOM (C++, University of Freiburg), WSL/Ubuntu, `cmake`/`make`

---

## Setup

```bash
git clone https://github.com/LabibaMenal/rajkot-transit-analyser.git
cd rajkot-transit-analyser

# data/rajkot_graph.geojson is already included, no build step needed

python3 -m http.server 8080
# open http://localhost:8080
```
It has to be served, not opened as a local file - the browser blocks the GeoJSON fetch under `file://` due to CORS.

---

## Deploy to GitHub Pages

```bash
git init
git add .
git commit -m "initial commit"
git remote add origin https://github.com/LabibaMenal/rajkot-transit-analyser.git
git push -u origin main
```
Then: repo → **Settings → Pages → Source: `main` branch → Save**.

---

## Known Limitations (Intentional Scope Cuts)

- **Straight-line edges, not real road curves.** No `shapes.txt` in the source feed, so every edge is a straight segment between two stops. Distances (and the Dijkstra results) are close but not road-accurate.
- **BFS features treat every edge as equal weight**, regardless of real distance - only the Dijkstra feature uses actual km. Fine for "is this stop reachable," not meant to be a real ETA.
- **Static schedule, not live data.** The GTFS feed is a published timetable, not GPS. There's no way currently to tell if a bus is actually running on time.
- **Everything computed client-side, on load.** Works fine at 716 stops / 903 edges; wouldn't scale as-is to a GTFS feed an order of magnitude larger without moving the graph prep server-side.

---

## What's Next

**On the dashboard itself:**
- Route filter - dropdown to isolate a single route's edges/stops
- Export report - dump the current simulation state (blocked roads, isolated stops, risk score) as a text summary
- Busiest-hour overlay - bucket `stop_times.txt` timestamps into hours and add a time slider; needs a small preprocessing step outside the browser, so this one's not a quick add
- Mobile-responsive layout - sidebar currently assumes desktop width

**On the data side** - bigger ideas from the original GTFS exploration that never got built:
- **Real-time GPS integration** - if RRL buses reported live location, actual arrivals could be diffed against `stop_times.txt`'s scheduled times, turning this from a structural snapshot into a live delay monitor
- **Ridership/demand overlay** - boarding counts per stop would show which corridors are over- or under-served, not just which are structurally fragile
- **Algorithmic route optimization** - use the same graph to suggest new routes that maximize how many residents fall within walking distance of a stop
- **Real road geometry** - add a proper `shapes.txt` so edges follow actual roads instead of straight lines; this is the single change that would most improve the Dijkstra distances
- **Weekday/weekend service split** - the current `calendar.txt` treats all 7 days identically, which likely doesn't match how RRL actually runs on Sundays
---
## Key Numbers

- 79 routes share a single road segment (highest in the network)
- 39 stops have degree 1 - zero redundancy, complete dependence on one route
- 82,603 stop-time records processed from the raw GTFS feed
- BFS complexity: O(V + E) = O(716 + 903), runs in under 2ms per query
- Dijkstra complexity: O(V²) = ~512K ops, under 1ms per query at this scale
---

## What I Learned

Working on this project taught me just looking for template tasks as a way of learning and clearing up some basic fundamental concepts is really redundant and definitely doesn't make me use the approach to my own understanding. Being provided with the exploratory project by my professor, Dr. Lakshay, helped me think about how i could optimise the routes and efficiency using the algorithms i was learning during that time. It also taught me how it could be taken a step further than what i was being assigned which helped me gain clarity, and definitely establish my fundamentals. While developing this made me realise how real-life problems can easily be tackled with these simple algorithms.

The exploratory half taught me that compiling a research tool from source in WSL and chaining three separate binaries together isn't really "beginner setup"- it's a real build-and-integration problem in its own right, distinct from anything I'd normally consider "the actual template project." The dashboard half taught me the opposite lesson: once you have stops-and-edges as a concept, a static renderer (LOOM/SVG) and an interactive one (Leaflet/BFS/Dijkstra) are the same underlying graph, just answering different questions - "what does the network look like" versus "what happens if I break this part of it." Rebuilding LOOM's output as GeoJSON instead of just displaying the SVG was the one decision that made the second question solvable at all.

---

## Project Structure

```
rajkot-transit-analyser/
├── index.html                 ← entire app (HTML + CSS + JS, single file)
├── data/
│   └── rajkot_graph.geojson   ← graph rebuilt from the LOOM pipeline output
└── README.md
```