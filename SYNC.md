# SYNC DOC — RajTransit Resilience Analyzer
> Hand this to the next Claude session to continue exactly from here.

---

## Project Status

COMPLETE AND DEPLOYABLE. All 4 features are working in index.html.

---

## What Is Built

Single file app: `index.html` + `data/rajkot_graph.geojson`

### Feature 1 — Vulnerability Heatmap ✅
- Edges colored green→red by number of routes sharing them
- Max load: 79 routes on one segment
- Sidebar: Risk Score, Top 10 overloaded corridors with route counts

### Feature 2 — Road Block Simulator ✅
- Click any edge → marks it blocked (red dashed)
- BFS (connected components) runs live to find isolated stops
- Sidebar shows: blocked count, routes disrupted, unreachable stops, % network intact
- Isolated stops turn red on map

### Feature 3 — Reachability Explorer ✅
- Click any stop → BFS expands outward up to N hops
- Slider: 2–20 hops (each hop ≈ 3 min of travel)
- Reachable stops rendered as cyan circles, fading by distance
- Sidebar: reachable count, % network coverage

### Feature 4 — Dead Zone Finder ✅
- All 39 degree-1 stops highlighted amber
- Sidebar lists all 39 stop names

### Feature 5 — Shortest Path (Dijkstra) ✅
- Click a start stop, then an end stop → Dijkstra finds the least-distance route
- Edge weight = real-world km, computed via Haversine formula summed along each edge's geometry (not fixed 1-per-hop like the BFS features)
- Path drawn as lime polyline; start = lime dot, end = orange dot
- Sidebar shows: start/end names, total distance (km), stop count on route
- Handles disconnected components gracefully ("No route found" message)
- Complexity: O(V²) plain min-scan (no heap) — 716² ≈ 512K ops, still instant for a single click

---

## Data Structure (GeoJSON)

```
Point features (716 stops):
  properties: { id: "0x...", station_label: "Stop Name", deg: 2 }
  geometry: { type: "Point", coordinates: [lon, lat] }

LineString features (903 edges):
  properties: {
    id: "0x...",
    from: "0x...",   ← matches Point id
    to: "0x...",     ← matches Point id
    lines: [{ label: "R9UP", color: "", id: "0x...", direction: "0x..." }, ...]
  }
  geometry: { type: "LineString", coordinates: [[lon,lat], ...] }
```

**Derived at runtime (not stored in the file):**
```
edgeWeights[idx] → km, computed once in init() via Haversine summed
                   over each edge's full coordinate list. Used only by
                   the Dijkstra Shortest Path feature; other features
                   still treat every edge as 1 hop.
```

---

## Files

```
rajtransit/
├── index.html              ← complete working app
├── data/
│   └── rajkot_graph.geojson
├── README.md
└── SYNC.md                 ← this file
```

---

## How to Run

```bash
cd rajtransit
python3 -m http.server 8080
# open localhost:8080
```

---

## How to Deploy (GitHub Pages)

```bash
git init && git add . && git commit -m "RajTransit v1"
git remote add origin https://github.com/LabibaPy/rajtransit.git
git push -u origin main
# then: repo Settings → Pages → main branch → Save
```

---

## Possible Improvements for Next Session

1. ~~**Dijkstra shortest path**~~ — ✅ DONE (see Feature 5 above). Built as its own mode `pathfinder`, weighted by real-world km via Haversine.

2. **Route filter** — dropdown to select one route, highlight only its edges and stops. Easy, 30 min.

3. **Export report** — button that generates a text summary of the current simulation state. Easy.

4. **Busiest hour overlay** — parse GTFS stop_times.txt into hourly buckets (preprocess.py), add time slider to show which edges are active at each hour. Hard, needs preprocessing.

5. **Mobile responsive** — sidebar collapses on small screens. Medium.

---

## Code Inventory — Feature 5 (Shortest Path)

New mode name: `pathfinder` (5th entry in the `modes` array in `setMode()` — order matters, it's index-matched to the mode buttons in the DOM).

New state fields on `S`: `pathStart`, `pathEnd` (both stop objects or `null`).

New global: `edgeWeights[]` (km per edge, computed once in `init()`), `pathLayer` (Leaflet layer group, like `reachLayer`/`deadLayer`).

New functions:
- `haversineKm(lat1,lon1,lat2,lon2)` — great-circle distance helper
- `onPathStopClick(stop)` — called from `onStopClick()` when mode is `pathfinder`; handles the 2-click start/end selection state machine
- `dijkstra(startId, endId)` — returns `{ path: [ids] | null, dist: km }`
- `runDijkstra()` — orchestrates solve + sidebar update
- `drawPath(path)` — renders the lime route line + start/end dots
- `clearPath()` / `resetPath(resetStatusText)` — cleanup, mirrors `clearReach()`/`clearDead()` pattern

New DOM ids: `sb-pathfinder`, `path-start`, `path-end`, `path-dist`, `path-hops`, `path-status`.

No changes to GTFS parsing, existing Features 1–4, or the GeoJSON schema. This was purely additive.

---

## Key Numbers for Resume/Interview

- 79 routes on single busiest edge (critical single point of failure)
- 39 stops with degree 1 (5.4% of network, zero redundancy)
- Risk Score computed as: (avg top-10 load / 168 routes) × 85 + (39/716) × 30
- BFS time complexity: O(V + E) = O(716 + 903) per query — near instant
- Total data processed: 82,603 stop-time records from GTFS
