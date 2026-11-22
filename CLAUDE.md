# CLAUDE.md

## Project
TIES (Tactical Intelligence Explainability System) – a semester MVP prototype. A tactical commander picks a start and target, gets 2 route options with an explainable risk comparison, approves via a KAMAN (intelligence officer) gate, reports field feedback with one tap, and a debrief updates trust weights. **All data is mock** (scenario: מבצע ״חומת אש״, Alpha → Bravo, 600 m urban).

Repo: https://github.com/bvidomlanski-crypto/TIES (public, branch `main`).

## Run
Open `index.html` in a browser. No build, no server, no dependencies, no network – keep it that way (offline-first is a product requirement). Do not add CDN scripts or external fonts/tiles.

## Structure
Everything lives in one file, `index.html` (inline CSS + JS), sectioned by `/* ==== ... ==== */` comments:
- **Map layer** – `MAP` is a 30×20 char grid (`#` building, `=` main street, `.` street, `c` courtyard, `J` jump house). `CELL = 20` (px and meters). `TER` holds per-terrain crossing time (`sec`) and line-of-sight exposure.
- **Threat data** – `TYPES` (kind: `point` / `los` / `area`), `BASE_THREATS` (T1–T8 relevant, T9–T16 deliberately filtered noise: out of area, expired >720 min, `dupOf`, empty), `LIVE_INTEL` (ingested by the 📡 button), `PRIOR` (Beta α/β for indicators and source types).
- **State** – global `S`; `S.persist` is saved to `localStorage` key `ties.mvp.v1` (bump the key if the persisted shape changes). All storage access is wrapped in try/catch.
- **Route engine** – `dijkstra()` 4-neighbour; `planRoutes()` = fast route + covered route (exposure penalty + penalty on route A cells), dropped if overlap ≥ 85%.
- **Trust/explainability engine** – `conf()` fuses sources (type trust × quality × age decay), `prob()` (indicator Beta mean or fixed), `exposure()`, `evalRoute()` (rifles cover top LOS threats ×0.25, engineering clears door IED ×0.15 +240 s), `evaluate()` picks `S.rec` by `risk + 0.15·time/maxTime`. Text: `recText()`, `routeWhy()`, `altSuggestion()`, `confInfo()`.
- **Feedback & sync** – `buildSegs()` (3 segments), `report()`, `trySync()` (Delta payload = unsynced reports only), `setOnline()`.
- **Debrief** – `processDebrief()` Bayesian update (point threats weight 1, LOS 0.25), `simulateNeighbors()`.
- **Render** – `renderHeader/Map/Panel`, `missionHTML()` switches on `S.stage`: `plan → pending → approved → moving → arrived`. Events via delegated `data-act` click handler.

## Conventions
- UI is Hebrew, RTL (`dir="rtl"`). In JS strings use Hebrew geresh/gershayim `׳` `״` (e.g. `קמ״ן`, `א׳`) instead of ASCII quotes to avoid escaping bugs.
- The fighter-facing UI must **never reveal source types** (VISINT/HUMINT/SIGINT/ALGO) – only abstract reliability ("מוצלב · N מקורות" / "מקור יחיד"). Source-type trust appears only in the debrief (KAMAN/server) view.
- Each displayed threat has exactly 3 reasons (`pattern` / `anomaly` / `terrain`).
- Colors are CSS tokens on `:root`; layout stacks under 900px; touch targets ≥ 44px.

## Expected demo numbers (sanity check after engine changes)
Default scenario, 3 rifles, no engineering: route A (main street) ≈ 56% risk / 4 min → **recommended**; route B (courtyards + jump house) ≈ 85% / 9 min. With engineering clearance: B ≈ 42% / 13 min → recommended. Filter shows 9 of 16 indications.

## Verify
Open in the browser pane and drive it via JS, e.g. click `[data-act=plan]`, `[data-act=approve]`, then inspect `S.rec`, `S.routes`, `S.filt`; run through `choose` → 3× `fb` → debrief `process`. Check for console errors.
