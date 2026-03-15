# Frontend Architecture (Original — Reference Only)

Detailed decomposition of how the original frontend works, for use as a spec. Not a codebase to extend.

## Data Flow

### Polling System (`useDataPolling.ts`)

Two-tier polling with adaptive timing:

| Tier | Endpoint | Startup | Steady State | Data |
|------|----------|---------|-------------|------|
| Fast | `/api/live-data/fast` | 3s | 15s | flights, ships, satellites, UAVs |
| Slow | `/api/live-data/slow` | 5s | 120s | news, earthquakes, GDELT, frontlines, stocks |

- ETag-based caching — `If-None-Match` header prevents re-downloading unchanged data (304 responses)
- Adaptive backoff — switches from startup to steady once `commercial_flights.length > 100`
- Uses `useRef` for data storage (persists across renders), `useState` for version bumps to trigger re-renders
- Returns `{ data, dataVersion, backendStatus }`

### Context Distribution (`DashboardDataContext.tsx`)

```typescript
interface DashboardDataContextValue {
    data: DashboardData;
    selectedEntity: { id, type, extra? };
    setSelectedEntity: (entity | null) => void;
}
```

Single provider wraps entire app. All components access via `useDashboardData()` hook.

### API Proxy (`app/api/[...path]/route.ts`)

Next.js catch-all route forwards `/api/*` to backend via `BACKEND_URL` env var (runtime-only, not `NEXT_PUBLIC_`). Enables Docker internal networking (`http://backend:8000`). Strips dangerous headers, decompresses gzip/br responses, returns 502 if upstream unreachable.

## Map Rendering

### Core Architecture

- **MapRef:** `useRef<MapRef>()` for imperative map control (flyTo, getBounds)
- **ViewState:** `useState<ViewState>` tracks zoom/lat/lng/bearing/pitch
- **MapBounds:** `[west, south, east, north]` with 20% buffer for viewport culling
- **ReadyState:** `mapReady` flag prevents operations before GL context ready

### Viewport Management

`updateBounds()` fires on `onIdle` and debounced `onMove` (500ms):
1. Reads raw bounds from map
2. Computes 20% expansion buffer for off-screen entity culling
3. POSTs unpadded bounds to `/api/viewport` every 1.5s for server-side AIS stream filtering

`inView(lat, lng)` — pure memoized function passed to all GeoJSON builders. Prevents off-screen entity rendering.

### Icon Loading

Two-phase load system:
1. **Critical icons** loaded synchronously on map ready (commercial/military aircraft, basic ships)
2. **Deferred icons** loaded in `setTimeout(..., 0)` (rare variants, military types, fire clusters)

All icons are SVG data URIs. `styleimagemissing` event listener retries pending images. `pendingImages` map prevents duplicate loads.

### Imperative Source Updates

GeoJSON sources are NOT updated via React props (too expensive for thousands of features). Instead:

```typescript
useImperativeSource(map, 'commercial-flights', commFlightsGeoJSON);
useImperativeSource(map, 'firms-fires', firmsGeoJSON, 2000);  // 2s debounce
```

Calls `source.setData(geojson)` directly on MapLibre source objects, bypassing React diffing.

## GeoJSON Builders

Each builder takes a data array + optional helpers (inView filter, interpolation functions) and returns a `FeatureCollection`.

### Flight Layer Builder

Reusable across commercial, private, military, tracked flights:

```typescript
interface FlightLayerConfig {
    colorMap: Record<string, string>;      // acType → iconId
    groundedMap: Record<string, string>;   // acType → grounded icon
    typeLabel: string;                     // 'flight', 'private_flight', etc.
    idPrefix: string;                      // 'flight-', 'pflight-', etc.
    milSpecialMap?: Record<string, string>; // military_type → icon override
    useTrackHeading?: boolean;              // prefer true_track over heading
}
```

Feature properties for every flight:
```typescript
{
    id: f.icao24 || f.callsign || `${idPrefix}${i}`,
    type: typeLabel,
    callsign: f.callsign || f.icao24,
    rotation: heading,
    iconId: /* from colorMap/groundedMap/milSpecialMap */
}
```

Icon selection logic:
1. If `milSpecialMap` and `f.military_type` matches → military override icon
2. If grounded (alt ≤ 100) → `groundedMap[acType]` (grey)
3. Otherwise → `colorMap[acType]`
4. Skip if `trackedIcaoSet.has(f.icao24)` (tracked flights render on separate layer)

### Other Builders

| Builder | Key Properties | Filtering |
|---------|---------------|-----------|
| Earthquakes | `id, type:'earthquake', name, title` | None |
| GPS Jamming | `severity, ratio, opacity (derived)` | None |
| CCTV | `id, type:'cctv', name, media_url, media_type` | `inView` |
| KiwiSDR | `id, type:'kiwisdr', name, url, users, bands, antenna` | `inView` |
| FIRMS Fires | `id, type:'firms_fire', frp, iconId (by FRP threshold)` | Clustered (radius=40) |
| Internet Outages | `id, type:'internet_outage', severity, region, country` | None |
| Data Centers | `id:'dc-{i}', type:'datacenter', name, company, address` | None |
| GDELT | `id, type:'gdelt', title` | `inView` |
| LiveUAMap | `id, type:'liveuamap', title, iconId (yellow/red)` | `inView` |
| UAVs | `id, type:'uav', callsign, rotation, iconId:'svgDrone'` | `inView` |
| Satellites | `id, type:'satellite', name, mission, sat_type, country, alt_km, iconId` | `inView` |
| Ships | `id:mmsi\|name, type:'ship', rotation, iconId (by ship type)` | Layer toggles + `inView` |
| Carriers | same as ships but `iconId:'svgCarrier'` | None |

### Fire Icon Thresholds (FIRMS)
```
FRP ≥ 100 MW → fire-darkred
FRP ≥ 20  MW → fire-red
FRP ≥ 5   MW → fire-orange
FRP < 5   MW → fire-yellow
```

## Interpolation System

Smooth motion between poll updates (data arrives every 15-60s, animation ticks every 250-500ms).

### Timer
```typescript
const [interpTick, setInterpTick] = useState(0);
useEffect(() => {
    const iv = setInterval(() => setInterpTick(t => t + 1), INTERP_TICK_MS);
    return () => clearInterval(iv);
}, []);
```

### Entity-Specific Interpolation

**Flights:** Uses speed_knots + (true_track || heading). Skips if grounded (alt ≤ 100) or speed ≤ 0. Waits 1s before starting interpolation.

**Ships:** Uses SOG (Speed Over Ground) + COG (Course Over Ground) from AIS. Falls back to heading if COG unavailable.

**Satellites:** Uses speed_knots + heading. Max clamp of 65 seconds (satellites don't update frequently).

### Position Math (`utils/positioning.ts`)

Great-circle interpolation using Haversine formula:
```
speedMps = speedKnots × 0.5144
distance = speedMps × clampedDt
R = 6,371,000 m (Earth radius)

newLat = asin(sin(lat)·cos(d/R) + cos(lat)·sin(d/R)·cos(heading))
newLng = lng + atan2(sin(heading)·sin(d/R)·cos(lat), cos(d/R) - sin(lat)·sin(newLat))
```

## Entity Selection & Click Handling

Single click handler routes by context:
1. If `measureMode` active → place waypoint
2. If entity already selected → deselect
3. If feature clicked:
   - Cluster → zoom in (+2 levels, 500ms animation)
   - Entity → extract `{id, type, name, media_url, extra: all_properties}`
4. Click on empty map → deselect

## Popup System

Each entity type gets its own popup renderer using `<Popup>` from react-map-gl. The popup follows the entity's position.

### Data Displayed Per Type

**Flight popup:** Callsign (colored by type), aircraft model, registration, airline (if commercial), altitude/speed/heading, route (origin → destination), wiki image

**Ship popup:** Name, type (carrier/tanker/yacht), MMSI, flag, SOG/COG, destination, wiki image (if carrier)

**Satellite popup:** Name, NORAD ID, type, country, mission (color-coded), altitude (km), wiki image

**UAV popup:** Callsign, model, UAV type, altitude/speed/heading, ICAO24, registration, squawk

**KiwiSDR popup:** Receiver name, location, active/max users, frequency bands, antenna type, action buttons (TRACK SIGNAL, PLAY)

**CCTV popup:** Camera name, live video player (HLS via hls.js or native Safari HLS)

## Icon System

### Aircraft Icons (`AircraftIcons.ts`)

Top-down silhouettes generated as SVG data URIs:
- **Airliner:** Wide swept wings, engine pods at wing roots
- **Turboprop:** Straight high wings, shorter fuselage
- **Bizjet:** Sleek small swept wings, T-tail
- **Helicopter:** Rotor disc representation

Color coding:
| Category | Color | Usage |
|----------|-------|-------|
| Commercial | Cyan | Default commercial flights |
| Private | Orange | General aviation |
| Jets | Purple | Business jets |
| Military | Yellow | Military classification |
| Grounded | Grey | Altitude ≤ 100m |
| Tracked | Alert color from DB | plane_alert_db matches |
| POTUS | Hot pink + golden halo | Hardcoded ICAO24 set |

### Satellite Icons (`SatelliteIcons.ts`)

Central square (satellite body) + side rectangles (solar panels) + center light dot.

Mission color coding:
| Mission | Color |
|---------|-------|
| military_recon | #ff3333 (red) |
| sar | #00e5ff (cyan) |
| sigint | #ffffff (white) |
| navigation | #4488ff (blue) |
| early_warning | #ff00ff (magenta) |
| commercial_imaging | #44ff44 (green) |
| space_station | #ffdd00 (gold) |

### Ship Icons

7 SVG variants: military (grey), carrier (special), cargo (yellow), tanker (yellow), passenger (blue), yacht (pink), default (white).

### Fire Cluster Icons

4 sizes based on `point_count`: sm (<10), md (<50), lg (<200), xl (≥200).

## Solar Terminator

Recomputed every 60 seconds. `computeNightPolygon()` generates ~360 vertices (1° longitude intervals). Rendered as fill layer with `fill-color: #0a0e1a`, `fill-opacity: 0.35`.

## Map Layer Structure (MapLibre)

Each entity type gets a `<Source>` + `<Layer>` pair:

```tsx
<Source id="commercial-flights" type="geojson" data={EMPTY_FC}>
    <Layer id="commercial-flights-layer" type="symbol"
        layout={{
            'icon-image': ['get', 'iconId'],
            'icon-allow-overlap': true,
            'icon-ignore-placement': true,
            'icon-rotate': ['get', 'rotation']
        }}
    />
</Source>
```

FIRMS fires use MapLibre's built-in clustering:
```tsx
<Source id="firms-fires" type="geojson" cluster={true} clusterRadius={40} clusterMaxZoom={10}>
    <Layer filter={['has', 'point_count']} ... />  {/* cluster icons */}
    <Layer filter={['!', ['has', 'point_count']]} ... />  {/* individual fires */}
</Source>
```

Cluster count labels rendered via separate `<Marker>` components on top.

## UI Panels

### Left Panel — Layer Controls (`WorldviewLeftPanel.tsx`)

22 layer toggles grouped by category. Each toggle controls visibility of a MapLibre layer.

Entity counts computed via `useMemo` — e.g., ship categories:
```typescript
for (const s of ships) {
    if (s.yacht_alert) trackedYacht++;
    else if (s.type === 'carrier' || s.type === 'military_vessel') military++;
    else if (s.type === 'tanker' || s.type === 'cargo') cargo++;
    else if (s.type === 'passenger') passenger++;
    else civilian++;
}
```

GIBS imagery: date slider + opacity control + play animation (increments date by 1 day every 1.5s, loops back 30 days).

POTUS fleet detection: hardcoded ICAO24 lookup against tracked_flights array.

Data freshness display: ISO timestamp per layer, rendered as relative time ("15s ago", "3m ago").

### News Feed (`NewsFeed.tsx`)

- Wikipedia image fetching: maps aircraft model codes → Wikipedia article titles → REST API thumbnail fetch
- Module-level cache persists across re-renders
- HLS video player: hls.js for Chrome/Firefox, native for Safari
- Risk score color coding on article cards
- Click article → flyTo location on map
- Media embedding: images, HLS video, iframes (no sandbox attributes)

### Radio Intercept (`RadioInterceptPanel.tsx`)

- Eavesdrop mode: click map → fetch `/api/radio/nearest?lat=&lng=` → find OpenMHz system → auto-play
- Scan mode: auto-cycle through nearby feeds
- Audio via imperative `HTMLAudioElement` management
- Failure states: "NO RECENT COMMS", "NO LOCAL REPEATERS FOUND", "ENCRYPTED / VOID"

### Find/Locate Bar (`FindLocateBar.tsx`)

- Nominatim geocoding for place search
- Coordinate parsing (decimal degrees, DMS)
- Callsign/vessel name search against current data

### Markets Panel (`MarketsPanel.tsx`)

Defense stocks + oil prices from slow-tier polling data.

## State Management Summary

| State | Location | Scope |
|-------|----------|-------|
| Polled data | `useRef` in `useDataPolling` | App-wide via context |
| Selected entity | `useState` in context | App-wide |
| Active layers | `useState` in `page.tsx` | Passed as props |
| Map view state | `useState` in `MaplibreViewer` | Map component |
| Map bounds | `useRef` in `MaplibreViewer` | Map component |
| Eavesdrop mode | `useState` in `page.tsx` | Passed as props |
| Measure mode | `useState` in `page.tsx` | Passed as props |
| GIBS date/opacity | `useState` in `page.tsx` | Passed as props |
| Theme/HUD color | `localStorage` via `ThemeContext` | App-wide |
| Admin key | `localStorage` (`sb_admin_key`) | Settings panel |
| Wiki image cache | Module-level `Map` | NewsFeed component |
| Pending map images | Module-level `Map` | MaplibreViewer |

## Key Performance Patterns (to keep)

1. **Imperative GeoJSON updates** — bypass React diffing on high-volume layers
2. **Viewport culling** — `inView()` filters features before GeoJSON creation
3. **ETag caching** — 304 responses skip deserialization and re-render
4. **Debouncing** — viewport updates (500ms), bounds posting (1.5s), search (1s)
5. **Progressive icon loading** — critical first, deferred in microtask

## Key Performance Problems (to fix)

1. **Full GeoJSON rebuild every tick** — should diff against previous state
2. **No list virtualization** — news feed, radio feeds render all items in DOM
3. **Interpolation computed for all entities** — should only interpolate visible ones
4. **2,302-line map component** — impossible to optimize pieces independently
5. **Module-level caches** — no eviction policy, grow unbounded
