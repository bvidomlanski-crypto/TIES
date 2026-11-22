# CLAUDE.md

## Project
TIES (Tactical Intelligence Explainability System) – a semester MVP prototype. A tactical commander picks a start and target, gets 2 route options with an explainable risk comparison, approves via a KAMAN (intelligence officer) gate, reports field feedback with one tap, and a debrief updates trust weights. **All threat data is mock.** Two areas (header selector):
- **schematic** – hand-made 30×20 grid, scenario מבצע ״חומת אש״ (Alpha → Bravo, 600 m).
- **khan** – real street/building layout of central Khan Younis from OpenStreetMap, with mock threats.

Repo: https://github.com/bvidomlanski-crypto/TIES (public). Work on `dev`, merge to `main` when stable.

## Run
Open `index.html` directly from disk. No build, no server, no npm, no CDN – keep it that way (offline-first is a product requirement). Real-map data is loaded with a plain `<script src="data/khan-younis.js">` (works on `file://`, no fetch).
- The optional satellite background (Esri World Imagery XYZ tiles as SVG `<image>`) is the only network use; it's off by default and hidden when the simulated network is offline.
- Claude's browser preview loads pages as `data:` URLs, so relative scripts don't load there – use `.claude/launch.json` (local static server, not committed) and `preview_start`.

## Structure
Everything is in `index.html` (inline CSS + JS), sectioned by `/* ==== ... ==== */` comments:
- **Terrain/threat types** – `TER_BASE` (per-terrain `spm` = seconds per meter, `exposed` = line-of-sight exposure; `#` building, `=` main road, `.` street, `c` courtyard/open, `J` jump house), `BREACH` (s to breach a building), `TYPES` (`kind` point/los/area, `rm` radius in meters), `PRIOR` (Beta α/β for indicators and source types).
- **Areas** – `AREAS[id] = {cell, map (array of strings), bid (building id per cell or null), breach, ter overrides, origin, dest, threats, live, labels, vector?, bbox?, mission}`. `setArea(id)` sets the active globals `A, MAP, W, H, CELL, BID, TER` and resets the mission.
  - Schematic: `SCHEM_MAP`, `SCHEM_THREATS` (T1–T8 relevant, T9–T16 deliberate noise), `SCHEM_LIVE`.
  - Khan Younis: `rasterize(D, 5)` turns OSM vectors into a 5 m grid (buildings via point-in-polygon, roads buffered by `ROAD_CLS` width). `KHAN_THREATS` (K1–K16) use `pos:[x,y]` in meters; `bld:true` binds a threat to the building under it (`t.bid`). `streetLabels()` + `STREET_HE` for Hebrew street names. Buildings are breachable only in the covered profile (`breach:true`).
- **State** – global `S`; `S.persist` (learning weights, reports, sync log, area, sat) in `localStorage` key `ties.mvp.v1` (bump if the shape changes incompatibly). Storage access is wrapped in try/catch.
- **Route engine** – `dijkstra()` with a binary heap on cell indices; `stepSec()` = terrain time + breach cost. `planRoutes()` = fast route A + covered route B (exposure penalty, penalty on A's cells, may breach buildings); B dropped if overlap ≥ 85%.
- **Trust/explainability engine** – `conf()` (source type trust × quality × age decay), `prob()`, `exposure()` (normalized per 20 m so cell size doesn't change risk), `filterThreats()`, `evalRoute()` (rifles cover top LOS threats ×0.25, engineering clears door IED ×0.15 +240 s), `evaluate()` picks `S.rec` by `risk + 0.15·time/maxTime`. Texts: `recText()`, `routeWhy()`, `altSuggestion()`, `confInfo()`.
- **Feedback & sync** – `buildSegs()`, `report()`, `trySync()` (Delta payload = unsynced reports only), `setOnline()`.
- **Debrief** – `processDebrief()` Bayesian update, `simulateNeighbors()`.
- **Render** – `renderMap()` draws cells as rects (schematic) or OSM polygons/polylines (vector areas); sizes scale with `U()`. `missionHTML()` switches on `S.stage`: `plan → pending → approved → moving → arrived`. Delegated `data-act` click handler.

## Data file
`data/khan-younis.js` is generated from OpenStreetMap (ODbL – keep the attribution). bbox S,W,N,E = 31.3416,34.2982,31.3474,34.3088 (~1008×641 m). Coordinates are local meters (x east, y south from the NW corner, equirectangular around the bbox centre). To regenerate: Overpass query `way["building"](bbox); way["highway"](bbox); out geom;` then convert to the same compact format (`buildings:[[x,y,...]]`, `roads:[{c:highway,n:name,p:[x,y,...]}]`). Changing the bbox/data moves the grid – re-check `KHAN_THREATS` positions and the default origin/dest.

## Conventions
- UI is Hebrew, RTL. In JS strings use `׳` `״` (e.g. `קמ״ן`, `א׳`) instead of ASCII quotes.
- The fighter-facing UI must **never reveal source types** (VISINT/HUMINT/SIGINT/ALGO) – only abstract reliability. Source-type trust appears only in the debrief (KAMAN/server) view.
- Each displayed threat has exactly 3 reasons (`pattern` / `anomaly` / `terrain`). Mock threats must stay clearly labelled as mock (attribution line under the map).
- Colors are CSS tokens on `:root`; layout stacks under 900px; touch targets ≥ 44px.

## Expected demo numbers (sanity check after engine changes)
3 rifles, no engineering:
- Schematic: A (main street) ≈ 56% / 4 min → **recommended**; B (courtyards + jump house) ≈ 85% / 9 min. With engineering: B ≈ 42% / 13 min → recommended. Filter shows 9 of 16.
- Khan Younis: A (Al-Bahr → Jamal Abd al-Nasser → Al-Sonya) ≈ 56% / 5 min → **recommended**; B (alleys + jump house K1) ≈ 90% / 11 min. With engineering: B ≈ 47% / 15 min (A still recommended on time). Filter shows 8 of 16. Planning ≈ 0.1 s.

## Verify
Start the preview server, open the page, and drive it via JS: `setArea('khan'); render();` then click `[data-act=plan]`, `[data-act=approve]`, inspect `S.rec`, `S.routes`, `S.filt`; run `choose` → 3× `fb` → debrief `process`. Check for console errors and that the schematic numbers above are unchanged.
