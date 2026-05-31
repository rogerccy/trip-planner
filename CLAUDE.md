# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development

No build system. The entire app is `index.html` — edit it directly and open `file:///path/to/trip-planner/index.html` in a browser to test. No server needed.

Deploy: `git add index.html && git commit && git push` from this directory. GitHub Pages auto-deploys from `main`.

Required GCP APIs: Maps JavaScript API, Places API (New), Routes API.

## Architecture

Single-file app: `<style>` → HTML body → `<script>`, following the same pattern as sibling projects in the workspace.

### State

Two module-level objects hold all state:
- `S` — settings: `{ apiKey, pat, gistId, activeTripId, colorblind }`
- `DATA` — trip data: `{ trips: [{ id, name, startDate, hotels[], days[] }] }`

`_`-prefixed fields (`_eff`, `_legs`, `_timeline`, `_expanded`, `_virtual`) are runtime-only caches — stripped from persistence via the `JSON.stringify` replacer. Never store user data in `_`-prefixed fields.

### Data model

```
trip: { id, name, startDate: 'YYYY-MM-DD', hotels: [], days: [] }

hotel: { id, name, address, lat, lng, placeId, phone, note, hours, nights: [dayIndex], legMode }

day: { id, label, departureTime: 'HH:MM', stops: [],
       _eff, _legs, _timeline }   ← runtime caches

stop: { id, name, address, lat, lng, placeId,
        legMode, transitModes: [], transitPref: '',
        dwellMinutes: 60, note: '', hours: null,
        _expanded }
```

### Hotel injection (`getEffectiveStops`)

Hotels are stored separately and injected as virtual first/last stops during render and route calculation — never stored in `day.stops`. A hotel appears as the first stop of day N if `hotel.nights.includes(N-1)`, and as the last stop if `hotel.nights.includes(N)`. Virtual stops have `_virtual: true` and `_hotelId`.

### Route calculation flow

1. `recalcDay(day, trip)` — rebuilds `day._eff` via `getEffectiveStops()`, then fans out `calcAndRenderLeg()` calls in parallel for all N-1 legs.
2. `calcAndRenderLeg(day, i, trip)` — calls Routes API v2, stores result in `day._legs[i]`, then calls `computeTimeline()` and triggers `renderSidebar()` + `drawPolylines()`.
3. Transit legs use a different field mask (no polyline): `routes.duration,routes.distanceMeters,routes.legs.steps.navigationInstruction,routes.legs.steps.staticDuration,routes.legs.steps.transitDetails`
4. Non-transit field mask includes `routes.polyline.encodedPolyline` for map rendering.

### Timeline computation

`computeTimeline(day, effectiveStops, legs)` — walks the effective stops array, tracking current time in minutes from midnight. Each stop contributes `dwellMinutes`; each leg contributes `leg.secs / 60`. Returns `[{ arrive, depart }]` parallel to `effectiveStops`.

### Leg editing state

`editingLeg` (module-level `let`) holds `'dayId-stopIndex'` when a mode picker is open. `toggleModePicker()` sets/clears it. `renderSidebar()` reads it to show the inline mode picker.

### APIs

- **Places API v1**: `POST https://places.googleapis.com/v1/places:searchText` with field mask `places.id,places.displayName,places.formattedAddress,places.location,places.regularOpeningHours,places.nationalPhoneNumber`
- **Routes API v2**: `POST https://routes.googleapis.com/directions/v2:computeRoutes` — field mask varies by transit vs non-transit (see above)
- **Gist sync**: paginated `GET /gists` to find by filename → PATCH/POST to update. File: `trip-planner.json`.

### Colorblind mode

`S.colorblind` toggles between `PALETTE_DEFAULT` and `PALETTE_CB` (Wong's 8-color palette). `dayColor(i)` selects the correct palette. Hours open/closed status uses shape+text (`✓`/`✕`), not color alone.

### localStorage keys

`tp_key`, `tp_pat`, `tp_gist`, `tp_active`, `tp_cb`, `tp_data`

### Taxi fare formula (Taiwan)

NT$85 base for first 1250m, NT$5 per 200m after. Implemented in `calcTaxiFare(meters)`.

### CSS conventions

Dark theme via `:root` variables (`--bg`, `--surface`, `--surface2`, `--surface3`, `--accent` #c8a96e, `--text`, `--sub`, `--muted`, `--muted2`, `--danger`, `--blue`, `--green`).

TripJam-style stop rows use `.tl-stop` / `.tl-dot` / `.tl-leg` / `.tl-leg-inner` classes. The inline mode picker uses `.mode-picker-inline` / `.mp-btn`. Transit sub-options use `.transit-sub` / `.t-btn`.
