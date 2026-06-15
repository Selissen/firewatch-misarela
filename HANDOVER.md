# Handover — Misárela Fire Watch: algorithm upgrade

**For:** a Claude Code agent implementing changes autonomously.
**Artifact:** a single static `index.html` (the dashboard). No build step, no backend.
**Goal:** improve the risk *algorithm* and data sources, in the priority order below, without breaking the "just open the page / serve on GitHub Pages" constraint.

---

## Progress — what is already done (do not redo)

| Commit | What landed |
|---|---|
| `chore: add CONFIG block + NOTES.md scaffold` | Top-level `CONFIG` object with `firmsMapKey`, `sources` flags, `bboxRadiusDeg`. `NOTES.md` scaffold. |
| `spike: record feed reachability/CORS (Task 0)` | All candidate feeds tested. Findings in `NOTES.md`. EFFIS blocked. IPMA RCM confirmed. |
| `feat: FIRMS key stored in localStorage, never in repo` | Password input inside "Adjust location" `<details>`. Key stored under `localStorage` key `mfw_firmsKey`, loaded into `CONFIG.firmsMapKey` on boot. Never in the repo. |
| `feat: FIRMS satellite hotspots, fused + labelled (Task 1)` | `getFirms()`, `clusterFirms()`, fused `renderFires(json, firmsClusters)`. Hollow map markers for satellite-only, dashed ring for sat-confirmed. Badges in fire list. 0.75 threat discount + one proxLevel step down for unconfirmed satellite fires. `refreshAll()` fetches fogos + FIRMS in parallel. Gated on `CONFIG.sources.firms && CONFIG.firmsMapKey`. |

**Next task to implement: Task 2 — IPMA RCM.** Start there. Do not re-run Task 0 checks; findings are already in `NOTES.md`.

---

## 1. Context you need

The app is a personal wildfire-monitoring dashboard for one hotel (Hotel Rural Misárela, Ferral / Montalegre, northern Portugal, edge of Peneda-Gerês). It helps the user decide whether to keep a booking (default 23–27 June) as conditions change day to day. It is served as a single static file on **GitHub Pages free tier** (public repo, HTTPS origin `https://<user>.github.io/<repo>/`).

### Hard constraints (do not violate)
- **Static only.** No server, no build, no bundler, no npm. One self-contained `index.html`. External libraries via CDN over **HTTPS only** (currently Leaflet + Google Fonts + CARTO tiles). Any `http://` resource will be blocked as mixed content on Pages — never introduce one.
- **No real secrets.** The only credential permitted in client code is a NASA FIRMS *map key* (low-sensitivity, rate-limited — already implemented via localStorage). Do not embed anything else.
- **No third-party CORS proxies.** If a feed won't do CORS from the browser, document it and fall back gracefully. Do not route traffic through public proxy services.
- **No fabricated data.** Missing value → render `—` / "unavailable". Never substitute a plausible default the user can't distinguish from real data.
- **Graceful degradation per panel.** Every fetch is wrapped in try/catch and fails *locally* — one dead feed must never blank the rest of the page.
- **Preserve the design system.** Reuse the existing CSS variables (`--clear/--monitor/--caution/--high`, `--river`, fonts `Space Grotesk / Inter / JetBrains Mono`). No new color or font systems.

### Critical testing note
Test from an HTTP(S) origin, never `file://`:
- Local: `python3 -m http.server 8000` then open `http://localhost:8000/index.html`.
- Or push to a branch and use the Pages deploy.

CORS behaviour observed on `file://` is meaningless.

### Current code map (single `<script>` IIFE in `index.html`)
- State: `HOTEL`, `CONFIG` (new), `LEVELS`, `FW_BANDS`, `LS='mfw_'`.
- Helpers: `clamp`, `toRad`, `compass`, `fmtDate`, `dParse`, `todayMid`, `dayKey`, `lsGet/lsSet`, `haversine(a,b)`, `bearing(from,to)`, `bandOf(score)`.
- Index: `fireWeather(tmax,rhmin,windmax,precip,droughtMult)`.
- Fires: `statusFactor(status)`, `getFirms()` (new), `clusterFirms(pts)` (new), `renderFires(json, firmsClusters)` (updated — accepts both sources).
- Weather: `getWeather()`, `computeDrought(d)`, `dailyHumidityMin()`, `renderWeather(wx)`.
- Air: `getAir()`, `renderAir(json)`.
- Verdict: `daysUntilCheckin()`, `proximityWeight(du)`, `setVerdict(wxR,fireR,airR)`.
- Trend: `recordSnapshot`, `renderTrend`, `drawSpark`.
- Orchestration: `refreshAll()` (fetches fogos + FIRMS in parallel); boot at bottom.

---

## 2. Remaining tasks, in order

### Task 2 — IPMA official fire risk (RCM) for Montalegre ← START HERE

**Status:** CORS confirmed (see `NOTES.md`). Montalegre DICO = `1706` (already resolved). Implement now.

**Endpoints (both confirmed CORS-readable, `Access-Control-Allow-Origin: *`):**
- Today's RCM: `https://api.ipma.pt/open-data/forecast/meteorology/rcm/rcm-d0.json`
- Tomorrow's RCM: `https://api.ipma.pt/open-data/forecast/meteorology/rcm/rcm-d1.json`
- `rcm-d2.json` returns 404 — only two days of data exist.

**Response shape** (confirmed in Task 0):
```json
{
  "dataPrev": "2026-06-15",
  "dataRun": "2026-06-14",
  "local": {
    "1706": { "data": { "rcm": 2 }, "dico": "1706", "latitude": 41.8245, "longitude": -7.79 }
  }
}
```
RCM class: integer 1–5 (Reduced → Maximum).

**Implement:**
- Add `CONFIG.ipmaMontalegreDico = '1706'` to the existing CONFIG block.
- Flip `CONFIG.sources.ipma = true` (it is currently `false`).
- `getIpmaRcm()` → fetches d0 and d1 in parallel; returns `{ 'YYYY-MM-DD': classN, ... }` for Montalegre only.
- Show the IPMA RCM read **alongside** (not replacing) the homemade index: for stay days within the 2-day RCM horizon, display the official class. For days beyond, the homemade index is the only read.
- **Where to show it:** best placed in the fire-weather day strip (`.day` cards). Each card already shows `score` and `band.label`. When the date matches an RCM day, add the official class as a secondary line — e.g. `IPMA RCM: High (3/5)`.
- **Calibration:** the provisional band mapping (in `NOTES.md`) is: RCM 1→Low, 2→Moderate, 3→High, 4→Very high, 5→Maximum. Use this to colour-code the IPMA line with existing `FW_BANDS` hex values. Do not invent new colours.
- **`CONFIG.sources.ipma` gate:** if `false`, `getIpmaRcm()` returns `{}` and nothing changes.
- Gate the fetch behind `CONFIG.sources.ipma` and wrap in try/catch; a failed IPMA fetch must not affect the weather panel.

**Acceptance:** when the stay (or today) is within the 2-day RCM window, each matching day card shows the official IPMA class alongside the heuristic score, visually distinct (labelled "IPMA RCM"), with colour from the calibrated band mapping.

**Commit message:** `feat: IPMA RCM official risk with fallback (Task 2)`

---

### Task 3 — Better threat model: continuous upwind + fire age/growth

**Implement:**
- **Continuous upwind factor.** Replace the binary flag: let θ = angle between "wind-blowing-toward" direction and the fire→hotel bearing; factor `= 1 + k·max(0, cos θ)·windSpeedScale·distanceScale`. Use **forecast** dominant wind for the stay dates when judging stay-relevant exposure, and **current** wind for the "now" read. Keep the existing `bearing()` helper.
- **Fire age / growth.** fogos.pt exposes a start time (`dateTime.sec`, or `date`+`hour`). Compute `ageHours`. Weight **young, active, resource-climbing** fires higher than old contained ones. Blend this with the resource proxy rather than replacing it; fall back to crew count when no timestamp.
- **Status robustness.** Keep keyword `statusFactor`, but if fogos provides a numeric status/`statusCode`, prefer it; default conservatively (treat unknown as active, not resolved). Add a comment that labels may change upstream.

**Acceptance:** upwind contribution varies smoothly with angle/speed/distance (no cliff at 55°); a fresh fast-growing fire outranks an older larger-resource one at similar distance; behaviour with missing timestamps is unchanged from today.

**Commit message:** `feat: continuous upwind + fire-age threat model (Task 3)`

---

### Task 4 — Forecast-based smoke + lead-time confidence

**Implement:**
- **Smoke outlook.** Extend `getAir()` to request the Open-Meteo air-quality **hourly forecast** (`pm2_5,pm10`, ~5-day horizon). If stay days fall inside the horizon, show a smoke outlook for them; otherwise show current + a "beyond forecast horizon" note.
- **Lead-time confidence.** Attach a confidence band to the fire-weather index and verdict by horizon: ≤3 days high, 4–7 medium, 8–16 low. Past ~5 days, render the index as a **range or with a low-confidence flag** — no false precision. The verdict copy should carry the confidence (e.g. "low-confidence at 11 days out").

**Acceptance:** stay-day smoke shows a forecast when in range and degrades honestly when not; the verdict and index visibly express confidence that decreases with lead time.

**Commit message:** `feat: forecast smoke + lead-time confidence (Task 4)`

---

### Task E — EFFIS / GWIS FWI forecast — **BLOCKED**

No key-free, browser-accessible JSON/WCS point query found. EFFIS exposes HTML dashboards only. Do not implement. Reason recorded in `NOTES.md`.

---

## 3. Cross-cutting "definition of done"
- Single static `index.html`, all-HTTPS, no mixed content, deploys unchanged to GitHub Pages.
- Every new source: try/catch, per-panel fallback, gated by a `CONFIG.sources` flag, and documented (latency, resolution, coverage, CORS status) both in a code comment and the in-page methodology.
- `localStorage` schema: handle old records missing new fields; bump `mfw_schema` if needed.
- The verdict blends forecast vs live by lead time; now also surfaces **confidence** and, where available, **official** (IPMA) numbers beside the heuristic.
- `NOTES.md` records every calibration mapping, with dates.
- Don't regress: location override, cancellation-window, seasonal strip, trend sparkline, auto-refresh, FIRMS satellite layer.

## 4. Commit sequence (remaining)
4. `feat: IPMA RCM official risk with fallback (Task 2)` ← next
5. `feat: continuous upwind + fire-age threat model (Task 3)`
6. `feat: forecast smoke + lead-time confidence (Task 4)`

Keep each commit independently shippable.

## 5. Honest limits to preserve (don't "fix" by faking)
- Distance stays straight-line; this is a gorge where terrain matters. Label it; don't imply spread modelling.
- Indices are decision aids, not safety guarantees. Keep the 112/ANEPC + "call the hotel" framing and the access-road (CM1397) evacuation note.
- If a feed is down or blocked, say so plainly — never paper over a gap with a default value.
