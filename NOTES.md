# NOTES — Misárela Fire Watch: algorithm upgrade

## Feed reachability / CORS (Task 0)

_Test date: 2026-06-15. Tested from `http://localhost:8000` (python3 -m http.server 8000). file:// findings are NOT recorded here — they are meaningless for CORS._

| Feed | Endpoint tested | HTTP status | CORS readable | Notes |
|---|---|---|---|---|
| NASA FIRMS VIIRS_NOAA20_NRT | `https://firms.modaps.eosdis.nasa.gov/api/area/csv/{key}/VIIRS_NOAA20_NRT/{bbox}/1` | — | — | Pending |
| NASA FIRMS VIIRS_SNPP_NRT | `https://firms.modaps.eosdis.nasa.gov/api/area/csv/{key}/VIIRS_SNPP_NRT/{bbox}/1` | — | — | Pending |
| IPMA admin areas | `https://api.ipma.pt/open-data/administrative-areas.json` | — | — | Pending |
| IPMA RCM today | `https://api.ipma.pt/open-data/forecast/meteorology/rcm/rcm-d0.json` | — | — | Pending |
| EFFIS/GWIS FWI | (searching for key-free browser-accessible point query) | — | — | Pending |

---

## Montalegre municipality code (Task 2)

Resolved from IPMA admin areas feed — to be filled in after Task 0.

---

## FW_BANDS ↔ IPMA RCM calibration (Task 2)

Mapping of the homemade fire-weather index bands to IPMA RCM classes, measured on overlapping days.

| RCM class | RCM label | Approx FW_BANDS score range | Notes |
|---|---|---|---|
| 1 | Reduced | — | To be calibrated |
| 2 | Moderate | — | To be calibrated |
| 3 | High | — | To be calibrated |
| 4 | Very high | — | To be calibrated |
| 5 | Maximum | — | To be calibrated |

---

## Blocked sources

_(None yet. Add here if a Task 0 verify step fails.)_
