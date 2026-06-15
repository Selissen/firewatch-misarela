# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Single-file static web app — `index.html` — that monitors wildfire risk for Hotel Rural Misárela (Ferral, Montalegre, Portugal). No build step, no package manager, no tests. Open the file directly in a browser to run it.

## Architecture

All logic lives inside a single IIFE in the `<script>` tag at the bottom of the file. There are no modules or imports — everything is plain ES2020+ vanilla JS.

### Data flow

`refreshAll()` orchestrates three parallel fetches → each renderer updates the DOM and returns a risk level → `setVerdict()` blends them into a final 0–3 verdict:

| Fetcher | API | Returns |
|---|---|---|
| `getWeather()` | Open-Meteo forecast + 31 past days | `renderWeather()` → `wxLevel` 0–3 |
| `getFires()` | fogos.pt `/v2/incidents/active` | `renderFires()` → `proxLevel` 0–3 |
| `getAir()` | Open-Meteo air-quality | `renderAir()` → `smokeLevel` 0–3 |

### Verdict blending (`setVerdict`)

`forecastLevel` (fire weather for the user's stay dates) and `nowLevel` (max of fire proximity + smoke) are blended by `proximityWeight(daysUntilCheckin)`. Far out (≥8 days), forecast drives 80%; at check-in it flips to 90% live. A hard override escalates to ≥Caution when a high-threat upwind fire is within 3 days of arrival.

### Fire-weather index (`fireWeather`)

Custom 0–100 heuristic: normalised temperature + relative humidity + wind speed, penalised by precipitation, then scaled by a drought multiplier (derived from the past 30 days of rainfall). The multiplier range is 0.75–1.30.

### Threat ranking (`renderFires`)

Each fire from fogos.pt is scored: `distanceFactor × sizeFactor × statusFactor × upwindBonus`. Distance uses haversine; upwind is true if the fire's bearing toward the hotel is within 55° of the wind direction and <60 km away. The list is sorted by descending threat score.

### Persistence (localStorage)

Keyed with prefix `mfw_`. Keys used: `hotel` (custom lat/lng), `cancelby` (free-cancel date), `hist` (up to 30 daily snapshots for the trend chart).

## Design system

CSS custom properties defined in `:root`. Colour tokens used across the entire file:

- `--granite` / `--panel` — backgrounds
- `--mist` / `--mist-dim` / `--mist-faint` — text hierarchy
- `--clear` (#3fae7a) / `--monitor` (#c9b23e) / `--caution` (#e0902e) / `--high` (#e2543b) — risk levels
- `--river` (#5fb6c9) — accent / interactive

Fonts: Space Grotesk (display), Inter (body), JetBrains Mono (mono) — all loaded from Google Fonts.

Map: Leaflet 1.9.4 with CARTO dark_all tiles.
