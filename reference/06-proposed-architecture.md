# Proposed Architecture

Clean-room rebuild as a modular OSINT platform.

## Design Principles
1. **Pluggable collectors** — each data source is a self-contained module, enumerated and injected at startup
2. **Dual-model database** — Postgres for application integrity (relational) + Apache AGE for entity graph (openCypher) + PostGIS for spatial queries — one database, two query paradigms
3. **Event-driven** — collectors push to a bus, frontend subscribes via WebSocket/SSE
4. **API-first** — versioned REST API, every feature accessible programmatically
5. **MCP server** — Claude can query data, plot markers, draw shapes, generate reports
6. **Self-contained graph** — AGE provides first-class graph schema for all OSINT data; Kappa integration is optional federation, not a dependency

## System Components

```
                                    ┌─────────────┐
                                    │   Claude     │
                                    │  (via MCP)   │
                                    └──────┬───────┘
                                           │
┌──────────────┐    ┌──────────┐    ┌──────┴───────┐    ┌─────────────┐
│  Collectors   │───▶│ Event Bus │───▶│   API Server  │───▶│  Frontend   │
│  (pluggable)  │    │          │    │   (FastAPI)   │    │  (Next.js)  │
└──────────────┘    └────┬─────┘    └──────┬───────┘    └─────────────┘
                         │                 │
                    ┌────┴─────────────────┴────┐    ┌──────────────┐
                    │      PostgreSQL            │    │ Kappa Graph  │
                    │  ┌──────────┬───────────┐  │◄──▶│ (optional    │
                    │  │ Relational│  AGE      │  │    │  federation) │
                    │  │ (app)    │  (graph)  │  │    └──────────────┘
                    │  ├──────────┼───────────┤  │
                    │  │ PostGIS (spatial)     │  │
                    │  └──────────────────────┘  │
                    └────────────────────────────┘
```

## Collector Module Interface

Each collector is a Python module that implements a standard interface:

```python
class Collector(Protocol):
    name: str                          # unique identifier
    description: str                   # human-readable
    tier: Literal["fast", "slow"]      # polling cadence
    interval_seconds: int              # how often to poll
    required_credentials: list[str]    # env var names needed
    entity_types: list[str]            # what it produces

    async def collect(self) -> list[Event]:
        """Fetch and return normalized events."""
        ...

    async def health_check(self) -> bool:
        """Verify connectivity to upstream source."""
        ...
```

Collectors are discovered at startup (entry points or directory scan), validated for credentials, and registered with the scheduler. Missing credentials = collector disabled with a warning, not a crash.

## Event Schema

All collectors produce normalized events:

```python
@dataclass
class Event:
    id: str                    # deterministic (source + source_id)
    source: str                # collector name
    entity_type: str           # "flight", "vessel", "earthquake", etc.
    timestamp: datetime        # when the event occurred
    position: Position | None  # lat/lng/alt
    properties: dict           # source-specific data
    ttl: timedelta | None      # how long this event is relevant
```

## Database Schema

Single Postgres instance, three capabilities: relational (application), graph (OSINT data), spatial (PostGIS).

### Relational Layer — Application Integrity

Standard tables for things that need referential integrity, constraints, and transactional guarantees.

```sql
-- Collector registry and lifecycle
CREATE TABLE collectors (
    name TEXT PRIMARY KEY,
    enabled BOOLEAN DEFAULT true,
    tier TEXT NOT NULL CHECK (tier IN ('fast', 'slow')),
    interval_seconds INT NOT NULL,
    last_run TIMESTAMPTZ,
    last_status TEXT,
    config JSONB DEFAULT '{}'
);

-- Credential store (encrypted at rest)
CREATE TABLE credentials (
    key TEXT PRIMARY KEY,
    value_encrypted BYTEA NOT NULL,
    collector TEXT REFERENCES collectors(name)
);

-- User annotations (shapes, markers, notes drawn on map)
CREATE TABLE annotations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type TEXT NOT NULL CHECK (type IN ('marker', 'polygon', 'line', 'note')),
    geometry GEOMETRY(Geometry, 4326),
    label TEXT,
    style JSONB DEFAULT '{}',
    created_by TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Analysis reports (Claude-generated or manual)
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    entity_refs TEXT[],          -- graph vertex IDs referenced
    created_at TIMESTAMPTZ DEFAULT now()
);
```

### Graph Layer — OSINT Entity Model (Apache AGE)

All collected intelligence lives in the graph. Entities are vertices, relationships are edges. This is where the interesting queries happen.

```sql
-- Initialize AGE and graph_accel
CREATE EXTENSION IF NOT EXISTS age;
CREATE EXTENSION IF NOT EXISTS graph_accel;
LOAD 'age';
SET search_path = ag_catalog, "$user", public;
SELECT create_graph('osint');
```

#### Vertex Labels

```sql
-- Entity: anything with persistent identity that moves or transmits
-- Properties: entity_id (deterministic), entity_type, name, lat, lng, alt,
--             heading, speed, last_seen, first_seen, properties (JSONB)
-- Subtypes via entity_type: aircraft, vessel, satellite, radio_station,
--                           camera, data_center, carrier_group

-- Event: something that happened at a point in time
-- Properties: event_id (deterministic), event_type, timestamp, lat, lng,
--             severity, description, source_url, properties (JSONB)
-- Subtypes via event_type: earthquake, fire, news_article, gdelt_event,
--                          conflict_update, internet_outage, space_weather

-- Location: named place with stable identity
-- Properties: location_id, name, type (airport, port, city, region, strait,
--             data_center_site), lat, lng, country_code, properties (JSONB)

-- Source: data provenance (one per collector)
-- Properties: source_id, collector_name, tier, last_run, status
```

#### Edge Labels

```sql
-- Spatial/temporal relationships
--   OBSERVED_AT     Entity/Event → Location   {timestamp, confidence}
--   NEAR            Entity ↔ Entity           {distance_km, timestamp}
--                   (co-location within configurable threshold)
--   TRANSITED       Entity → Location         {entered_at, exited_at}

-- Structural relationships
--   PART_OF         Entity → Entity           {role}
--                   (vessel → carrier_group, aircraft → squadron)
--   OPERATES_FROM   Entity → Location         {relationship: homeport/base}
--   REPORTED_BY     Event → Source             {ingest_timestamp}

-- Correlation relationships (computed, not raw ingest)
--   RELATED_TO      Event ↔ Event             {correlation_type, confidence}
--                   (temporal proximity, spatial proximity, shared entities)
--   MENTIONS        Event → Entity            {confidence, context_snippet}
--   DETECTED_BY     Event → Entity            {sensor_type}
--                   (fire detected by satellite, outage detected by probe)
```

#### Edge Properties — Confidence Model

Following the Kappa pattern: edges carry a `confidence` float (0.0–1.0).

- **1.0** — direct observation (ADS-B position, AIS position report)
- **0.8–0.9** — derived with high certainty (flight route from callsign lookup)
- **0.5–0.7** — inferred (GDELT geocoding, news article geolocation)
- **0.1–0.4** — speculative (carrier position from news scraping, correlation)
- **NULL/NaN** — confidence not applicable (structural relationships like PART_OF)

This enables confidence-filtered traversal: "show me only high-confidence connections."

#### Vertex Property Shapes

```json
// Entity vertex (aircraft example)
{
  "entity_id": "adsb:a88467",
  "entity_type": "aircraft",
  "name": "N648QX",
  "callsign": "QXE2192",
  "lat": 59.006653,
  "lng": -141.181549,
  "alt": 10972.8,
  "heading": 300.66,
  "speed_knots": 382.4,
  "model": "E75L",
  "icao24": "a88467",
  "squawk": "6631",
  "classification": "commercial",
  "last_seen": "2026-03-14T23:18:00Z",
  "first_seen": "2026-03-14T22:45:00Z"
}

// Entity vertex (vessel example)
{
  "entity_id": "ais:244670583",
  "entity_type": "vessel",
  "name": "EVER GIVEN",
  "mmsi": "244670583",
  "ship_type": 70,
  "ship_type_name": "cargo",
  "lat": 29.9167,
  "lng": 32.5833,
  "heading": 180,
  "speed_knots": 12.5,
  "destination": "ROTTERDAM",
  "last_seen": "2026-03-14T23:15:00Z"
}

// Event vertex (earthquake example)
{
  "event_id": "usgs:us7000abc1",
  "event_type": "earthquake",
  "timestamp": "2026-03-14T22:30:00Z",
  "lat": 38.7749,
  "lng": -122.4194,
  "magnitude": 4.2,
  "depth_km": 8.5,
  "description": "4.2 magnitude earthquake near San Francisco, CA",
  "source_url": "https://earthquake.usgs.gov/..."
}

// Event vertex (news article example)
{
  "event_id": "rss:sha256:a1b2c3d4",
  "event_type": "news_article",
  "timestamp": "2026-03-14T21:00:00Z",
  "lat": 48.8566,
  "lng": 2.3522,
  "headline": "NATO exercises begin in Baltic Sea",
  "source": "Reuters",
  "risk_score": 0.7,
  "source_url": "https://reuters.com/..."
}
```

### Graph Accelerator (graph_accel)

The Rust-based AGE accelerator plugin from the Kappa project deploys directly into this platform's Postgres instance. It maintains an in-memory adjacency list (HashMap-based, bidirectional) and provides sub-millisecond graph traversals where AGE's Cypher-to-SQL translation would take seconds or hang.

#### Configuration

```sql
-- GUC parameters (postgresql.conf or SET)
SET graph_accel.source_graph = 'osint';          -- graph to accelerate
SET graph_accel.node_id_property = 'entity_id';  -- app-level ID property
SET graph_accel.max_memory_mb = 4096;            -- per-backend memory cap
SET graph_accel.node_labels = '*';               -- load all vertex types
SET graph_accel.edge_types = '*';                -- load all edge types
SET graph_accel.auto_reload = true;              -- reload on stale detection
SET graph_accel.reload_debounce_sec = 5;         -- min time between reloads
```

#### Accelerated Operations

```sql
-- BFS neighborhood: "what's within 3 hops of this carrier group?"
SELECT * FROM graph_accel_neighborhood(
  'ais:carrier-cvn78',  -- start vertex (by app_id)
  3,                     -- max depth
  'both',                -- direction (outgoing/incoming/both)
  0.5                    -- min confidence filter (NULL = no filter)
);
-- Returns: node_id, label, app_id, distance, path_types[], path_directions[]
-- Performance: sub-millisecond vs AGE hanging at depth 3+

-- Shortest path: "how is this news event connected to that vessel?"
SELECT * FROM graph_accel_path(
  'rss:sha256:a1b2c3d4',  -- from
  'ais:244670583',          -- to
  10,                       -- max hops
  'both',                   -- direction
  NULL                      -- no confidence filter
);
-- Returns: ordered path steps with node IDs, labels, relationship types

-- K-shortest paths: "show me 5 different connection routes"
SELECT * FROM graph_accel_paths(
  'rss:sha256:a1b2c3d4', 'ais:244670583',
  10, 5, 'both', NULL
);
-- Returns: multiple alternative paths (Yen's algorithm)

-- Hub detection: "which entities have the most connections?"
SELECT * FROM graph_accel_degree(50);
-- Returns: top 50 vertices by total degree (in + out edges)

-- Subgraph extraction: "give me the full neighborhood graph"
SELECT * FROM graph_accel_subgraph(
  'ais:carrier-cvn78', 3, 'both', 0.5
);
-- Returns: all edges within the BFS neighborhood (for visualization/export)

-- Cache management
SELECT * FROM graph_accel_status();
-- Returns: node_count, edge_count, memory_bytes, is_stale, loaded_generation

SELECT graph_accel_invalidate('osint');
-- Bumps generation counter after bulk ingest, triggers async reload
```

#### Performance Profile

From Kappa benchmarks (236 nodes, ~120 edge types):

| Depth | AGE (Cypher→SQL) | graph_accel | Speedup |
|-------|-----------------|-------------|---------|
| 1 | 3,644ms | 0.101ms | 36,000x |
| 2 | 471ms | 0.066ms | 7,100x |
| 3 | 1,790ms | 0.122ms | 14,700x |
| 4 | 11,460ms | 0.267ms | 42,900x |
| 5 | 92,474ms | 0.378ms | 244,600x |
| 6 | Hangs | 0.377ms | -- |

At OSINT scale (thousands of entities, tens of thousands of edges), multi-hop queries like "what's connected to this carrier group within 4 hops" would be unusable without the accelerator.

#### Cache Invalidation Pattern

After collectors ingest new data into the graph:

```python
# In the API/collector layer (Python)
async def after_ingest(collector_name: str):
    # 1. Bulk insert vertices/edges via Cypher
    await db.execute_cypher("MERGE (e:Entity {entity_id: $id}) SET e += $props", ...)

    # 2. Invalidate accelerator cache (non-blocking)
    await db.execute("SELECT graph_accel_invalidate('osint')")

    # 3. Next read query auto-reloads from AGE tables
    #    Stale reads are acceptable — topology changes slowly relative to position updates
```

#### Python Facade (ported from Kappa)

```python
class GraphFacade:
    """Unified graph traversal — uses graph_accel when available, falls back to Cypher."""

    async def neighborhood(self, entity_id: str, max_depth: int = 2,
                           direction: str = 'both', min_confidence: float | None = None):
        if self._accel_ready:
            return await self._execute(
                "SELECT * FROM graph_accel_neighborhood(%s, %s, %s, %s)",
                (entity_id, max_depth, direction, min_confidence))
        return await self._neighborhood_cypher(entity_id, max_depth)

    async def find_path(self, from_id: str, to_id: str, max_hops: int = 10):
        ...

    async def find_paths(self, from_id: str, to_id: str, max_hops: int = 10, k: int = 5):
        ...

    async def degree(self, top_n: int = 100):
        ...

    async def subgraph(self, start_id: str, max_depth: int = 3,
                       direction: str = 'both', min_confidence: float | None = None):
        ...

    async def invalidate(self):
        return await self._execute("SELECT graph_accel_invalidate('osint')")
```

### Spatial Layer — PostGIS

PostGIS extends both the relational and graph layers.

```sql
-- Spatial index on annotation geometries (relational layer)
CREATE INDEX idx_annotations_geom ON annotations USING GIST (geometry);

-- Hybrid SQL+Cypher spatial queries
-- "Find all Entity vertices within 50km of a point"
SELECT * FROM cypher('osint', $$
  MATCH (e:Entity)
  WHERE e.lat IS NOT NULL
  RETURN e.id, e.name, e.entity_type, e.lat, e.lng
$$) AS (id agtype, name agtype, etype agtype, lat agtype, lng agtype)
WHERE ST_DWithin(
  ST_SetSRID(ST_MakePoint(lng::float, lat::float), 4326)::geography,
  ST_SetSRID(ST_MakePoint(-71.0, 42.3), 4326)::geography,
  50000  -- 50km
);

-- Viewport bounding box query (used by frontend map)
-- Combines graph_accel for relationship data with PostGIS for spatial filtering
SELECT * FROM cypher('osint', $$
  MATCH (e:Entity)
  WHERE e.lat IS NOT NULL
  RETURN e.id, e.entity_id, e.entity_type, e.name, e.lat, e.lng, e.properties
$$) AS (id agtype, eid agtype, etype agtype, name agtype, lat agtype, lng agtype, props agtype)
WHERE lat::float BETWEEN 40.0 AND 50.0
  AND lng::float BETWEEN -5.0 AND 10.0;
```

## API Design

```
-- Entities and Events --
/v1/entities                  GET    — paginated entity list (cursor-based)
/v1/entities/{id}             GET    — single entity detail + current position
/v1/entities/bbox             GET    — spatial query (lat/lng bounds + entity_types filter)
/v1/entities/stream           WS     — real-time entity position push (WebSocket)
/v1/events                    GET    — paginated event list (cursor-based)
/v1/events/{id}               GET    — single event detail
/v1/events/bbox               GET    — spatial query (lat/lng bounds + event_types filter)
/v1/events/stream             WS     — real-time event push (WebSocket)

-- Graph traversal (backed by graph_accel) --
/v1/graph/neighborhood/{id}   GET    — BFS neighborhood (params: depth, direction, min_confidence)
/v1/graph/path                GET    — shortest path between two entities (params: from, to, max_hops)
/v1/graph/paths               GET    — k-shortest paths (params: from, to, max_hops, k)
/v1/graph/hubs                GET    — top entities by degree centrality (params: top_n)
/v1/graph/subgraph/{id}       GET    — full edge set within neighborhood (for visualization)
/v1/graph/status              GET    — accelerator cache health

-- Collectors --
/v1/collectors                GET    — list registered collectors + status
/v1/collectors/{name}         GET    — collector detail + health
/v1/collectors/{name}/toggle  POST   — enable/disable

-- Map and UI --
/v1/layers                    GET    — available map layers + legend
/v1/reports                   GET    — generated analysis reports
/v1/reports                   POST   — create report (Claude or manual)
/v1/annotations               GET    — user-drawn shapes, markers, notes
/v1/annotations               POST   — create annotation
/v1/annotations/{id}          DELETE — remove annotation
```

## MCP Server Interface

Tools exposed to Claude:

```
-- Query --
query_entities(bbox, entity_types, time_range)   — search entities by area/type/time
query_events(bbox, event_types, time_range)      — search events by area/type/time
get_entity(id)                                    — entity detail + current position

-- Graph traversal --
neighborhood(entity_id, depth, min_confidence)    — BFS neighborhood of an entity
find_path(from_id, to_id)                         — shortest path between two entities
find_paths(from_id, to_id, k)                     — k alternative connection routes
get_hubs(top_n)                                    — most connected entities
get_subgraph(entity_id, depth)                     — full edge set for visualization

-- Map interaction --
plot_marker(lat, lng, label, style)               — add marker to map
draw_polygon(coordinates, label, style)           — draw shape on map
draw_line(coordinates, label, style)              — draw line on map
highlight_entities(ids, style)                    — highlight existing entities
get_annotations()                                  — read user-drawn shapes
clear_annotations(ids)                             — remove shapes

-- Reports --
post_report(title, body, entity_refs)             — publish analysis report
                                                     (also creates graph edges from
                                                     report → referenced entities)
```

## Kappa Graph Federation (Optional)

This platform is self-contained with its own AGE graph. Kappa integration is optional federation — syncing OSINT entities into the broader knowledge graph for cross-domain reasoning.

When enabled, a federation sync pushes graph vertices/edges to Kappa's API, establishing bidirectional references:

```
POST /api/concepts
{
  "type": "osint_event",
  "source": "shadowbroker",
  "external_ref": "sb://events/{id}",    # bidirectional link
  "properties": { ... },
  "relationships": [
    {"type": "located_at", "target": "geo://48.8566,2.3522"},
    {"type": "involves", "target": "entity://carrier/CVN-78"}
  ]
}
```

The Kappa Rust AGE accelerator plugin can also be used here — the same graph acceleration that powers Kappa's traversals works on this platform's AGE instance since it's the same underlying storage model. Shared tooling, independent deployments.

## Frontend Architecture

- Next.js with modular layer components (one per entity type)
- MapLibre GL with incremental GeoJSON diffing (not full rebuild)
- WebSocket subscription for real-time updates (no polling for fast-tier)
- List virtualization for news/radio/entity panels
- Dark mode with muted palette (not edgelord black)
- Annotation tools: draw polygons, place markers, add notes
- Report viewer: Claude-generated analysis rendered inline

## Secrets Management

- No `.env` files for API tokens in production
- Docker secrets or Vault for deployment
- Encrypted credential store in Postgres for runtime
- Collector modules declare required credentials; missing = disabled, not crashed
