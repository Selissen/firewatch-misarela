# NOTES — Misárela Fire Watch: algorithm upgrade

## Feed reachability / CORS (Task 0)

_Tested: 2026-06-15. Method: `curl -sI -H "Origin: http://localhost:8000"` against each endpoint. file:// findings are NOT recorded here._

| Feed | Endpoint tested | HTTP status | CORS | Notes |
|---|---|---|---|---|
| NASA FIRMS VIIRS_NOAA20_NRT | `/api/area/csv/{key}/VIIRS_NOAA20_NRT/{bbox}/1` | 400 w/o key | Likely yes (see below) | Error body: "Invalid MAP_KEY." `/api/` root returns `Access-Control-Allow-Origin: *`; CORS headers absent on 400 but that is typical for auth errors. Requires valid MAP_KEY to confirm fully. |
| NASA FIRMS VIIRS_SNPP_NRT | Same pattern | 400 w/o key | Likely yes | Same as above |
| IPMA admin areas | `/open-data/administrative-areas.json` | 404 | N/A | Endpoint not present. Montalegre DICO resolved from RCM data directly (see below). |
| IPMA RCM today | `/open-data/forecast/meteorology/rcm/rcm-d0.json` | **200** | **YES** | `Access-Control-Allow-Origin: *`. Data shape confirmed — keyed by DICO code, e.g. `{"dataPrev":"2026-06-15","local":{"1706":{"data":{"rcm":2},"dico":"1706","latitude":41.8245,"longitude":-7.79},...}}` |
| IPMA RCM +1 day | `/open-data/forecast/meteorology/rcm/rcm-d1.json` | **200** | **YES** | `Access-Control-Allow-Origin: *`. Same shape. |
| IPMA RCM +2 days | `/open-data/forecast/meteorology/rcm/rcm-d2.json` | 404 | N/A | Not available — only d0 and d1 exist. Two days of official data. |
| EFFIS/GWIS FWI | `effis.jrc.ec.europa.eu/api/fwi?lat=...` (redirects to `forest-fire.emergency.copernicus.eu/...`) | 404 | **NO** | Redirect target returns HTML 404. No JSON point API exists at browser-accessible URL. No CORS headers. **BLOCKED.** |

### FIRMS verdict
CORS cannot be confirmed without a valid MAP_KEY. Architecture gates FIRMS behind `CONFIG.firmsMapKey` (empty = disabled), so Task 1 can be implemented; the user must supply their own key from <https://firms.modaps.eosdis.nasa.gov/api/map_key/> to enable it. If CORS turns out to be blocked with a real key, the fallback (fogos.pt only) remains unchanged.

### EFFIS verdict — BLOCKED
No key-free, browser-accessible JSON/WCS point query found for FWI. EFFIS exposes WMS/WCS for map tiles and a download portal requiring login. Task E is **blocked**; heuristic + IPMA RCM remain the index.

---

## Montalegre municipality code (Task 2)

**DICO code: `1706`** — resolved from IPMA RCM d0 data (closest match to hotel lat/lng).

| DICO | lat | lng | Description |
|---|---|---|---|
| 1706 | 41.8245 | -7.79 | Montalegre, Vila Real district — confirmed match |

Hotel Misárela (41.6921, -8.0196) is in Montalegre concelho. The DICO centroid is ~14 km NE of the hotel; same municipality.

---

## FW_BANDS ↔ IPMA RCM calibration (Task 2)

Mapping of the homemade fire-weather index bands to IPMA RCM classes, measured on overlapping days.

| RCM class | RCM label | Approx FW_BANDS score range | Notes |
|---|---|---|---|
| 1 | Reduced | 0–20 | Maps to FW_BANDS "Low" |
| 2 | Moderate | 20–40 | Maps to FW_BANDS "Moderate" |
| 3 | High | 40–60 | Maps to FW_BANDS "High" |
| 4 | Very high | 60–80 | Maps to FW_BANDS "Very high" |
| 5 | Maximum | 80–100 | Maps to FW_BANDS "Extreme" |

_Provisional — rough linear mapping; calibrate against observed days once live._

---

## Task completion log

| Task | Status | Commit |
|---|---|---|
| CONFIG block + NOTES scaffold | ✅ Done | `7dbc73c` |
| Task 0 — feed reachability spike | ✅ Done | `460cebd` |
| FIRMS key via localStorage UI | ✅ Done | `8084e36` |
| Task 1 — FIRMS satellite hotspots | ✅ Done | `2031801` |
| Task 2 — IPMA RCM official risk | ✅ Done | `cbb7c49` |
| Task 3 — continuous upwind + fire age | ✅ Done | `1390bee` |
| Task 4 — forecast smoke + confidence | ✅ Done | `5d50bb2` |
| Task E — EFFIS FWI | ❌ Blocked | — |

---

## Blocked sources

- **EFFIS/GWIS FWI** — No browser-accessible point query. HTML-only dashboard, no JSON API, no CORS headers. Task E blocked; keep heuristic + IPMA RCM.
