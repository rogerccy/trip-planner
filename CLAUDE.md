# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development

No build system. The entire app is `index.html` — edit it directly and open `file:///path/to/trip-planner/index.html` in a browser to test. No server needed.

Deploy: `git add index.html && git commit && git push` from this directory. GitHub Pages auto-deploys from `main` → https://rogerccy.github.io/trip-planner/

Required GCP APIs: Maps JavaScript API, Places API (New), Routes API.

## Architecture

Single-file app: `<style>` → HTML body → `<script>`.

### State

```js
S    = { apiKey, pat, gistId, activeTripId, colorblind }
DATA = { trips: [{ id, name, startDate, hotels[], days[] }] }
selectedPanel = null  // { type:'stop'|'leg'|'hotel', key, dayId?, stopId?, legIdx?, hotelId? }
```

`_`-prefixed fields are runtime caches. `save()` strips `_virtual`, `_hotelId`, `_eff`, `_timeline`, `_expanded` from localStorage but **keeps `_legs`** as a route cache. `pushGist()` strips all `_`-prefixed fields for clean cloud backup.

### Data model

```
trip:  { id, name, startDate: 'YYYY-MM-DD', hotels: [], days: [] }
hotel: { id, name, address, lat, lng, placeId, phone, note, nights: [k], legMode }
day:   { id, label, departureTime: 'HH:MM', stops: [], _eff, _legs, _timeline }
stop:  { id, name, address, lat, lng, placeId, legMode, transitModes: [],
         transitPref: '', dwellMinutes: 60, note: '', hours: null }
```

### Hotel night index (1-based)

`hotel.nights` stores 1-based indices: Night k = between Day k and Day k+1 (1-indexed). For N days, valid nights are 1 through N-1.

In `getEffectiveStops(day, dayIndex, trip)` (0-indexed `dayIndex`):
- **Prepend** (hotel at start of day): `nights.includes(dayIndex)` — guarded by `if (dayIndex > 0)`
- **Append** (hotel at end of day): `nights.includes(dayIndex + 1)`

Virtual hotel stops have `_virtual: true` and `_hotelId`. `boot()` migrates stale night index 0 from old data.

### Route calculation flow

1. `recalcDay(day, trip)` — rebuilds `day._eff` via `getEffectiveStops()`, fans out `calcAndRenderLeg()` for all N-1 legs in parallel. Sets `day._legs = new Array(n).fill(null)` first.
2. `calcAndRenderLeg(day, i, trip)` — calls Routes API v2, stores result in `day._legs[i]`, calls `computeTimeline()`, triggers `renderSidebar()` + `drawPolylines()` + `renderDetailPanel()`.
3. Transit field mask omits polyline; non-transit includes `routes.polyline.encodedPolyline`.
4. On boot: if `day._legs` exists in localStorage, rebuild `_eff` and `_timeline` locally without API calls. Only recalc when `day._legs === null`.

### Timeline

`computeTimeline(day, effectiveStops, legs)` — walks stops tracking minutes from midnight. Each stop adds `dwellMinutes`; each leg adds `leg.secs / 60`. Returns `[{ arrive, depart }]` parallel to `effectiveStops`.

### Panel system

`selectedPanel` drives the slide-in `#detail-panel` (width: 0 → 300px transition). Three types:
- `selectStop(dayId, stopId)` → stop details (location, arrive/stay/leave, hours, notes)
- `selectLeg(dayId, legIdx)` → leg details (mode picker, transit sub-options, route steps)
- `selectHotel(hotelId)` → hotel details (address, phone, notes, delete)
- `closePanel()` clears state and collapses panel

`switchTab()` closes the panel when switching between itinerary and hotels tabs.

### Gist sync

- **Boot / settings save**: `pullGist()` — fetches remote, overwrites local, re-renders
- **Every edit**: `save()` → `pushGist()` — auto-push after localStorage write
- **Sync button**: `syncGist()` → delegates to `pushGist()`

### APIs

- **Places API v1**: `POST https://places.googleapis.com/v1/places:searchText`
- **Routes API v2**: `POST https://routes.googleapis.com/directions/v2:computeRoutes`
- **Gist**: paginated `GET /gists` to find by filename `trip-planner.json` → PATCH/POST

### UI conventions

- All icons: Google Material Icons (`<span class="material-icons">name</span>`). Helper: `mi(name, size)`.
- Stop rows: `.tl-stop` / `.tl-dot` (16px circle) / `.tl-leg` / `.tl-leg-inner`. Hover reveals `.tl-stop-actions` (copy + delete).
- Hotel cards: `.hotel-card` / `.hotel-card-row` with inline night buttons (`.night-btn`). Hover reveals `.hotel-card-actions`.
- Dark theme CSS vars: `--bg`, `--surface`, `--surface2`, `--surface3`, `--accent` (#c8a96e), `--text`, `--sub`, `--muted`, `--muted2`, `--danger`, `--blue`, `--green`.

### localStorage keys

`tp_key`, `tp_pat`, `tp_gist`, `tp_active`, `tp_cb`, `tp_data`

### Taxi fare (Taiwan)

NT$85 base for first 1250m, NT$5 per 200m after — `calcTaxiFare(meters)`.
