# Temporal Model & Analysis

How the graph keeps itself straight over time.

## The Problem

OSINT data has fundamentally different temporal characteristics:

| Data Type | Temporal Nature | Example |
|-----------|----------------|---------|
| Entity position | Continuously changing | Aircraft at (lat, lng) right now |
| Observation | Point-in-time fact | "This vessel was HERE at 14:32:07 UTC" |
| Event | Instantaneous or bounded | Earthquake at T, conflict update from T1 to T2 |
| Relationship | Valid for a window | "These two ships were NEAR each other from T1 to T2" |
| Analysis/Report | Snapshot conclusion | "At epoch 47201, these 3 events appear correlated" |

A naive graph that overwrites entity positions loses history. A graph that keeps everything grows unbounded. We need both: current state for the map, historical state for analysis.

## Epoch Model

An **epoch** is a monotonically increasing integer that advances every time the graph state changes materially. It's not a timestamp — it's a logical clock that gives ordering guarantees independent of wall-clock drift.

```sql
-- Relational table (application layer)
CREATE TABLE epochs (
    epoch BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at TIMESTAMPTZ,                -- NULL = current epoch
    collector TEXT,                       -- which collector triggered this epoch
    entity_count INT,                     -- snapshot stats
    edge_count INT,
    notes TEXT                           -- optional: "bulk ingest from GDELT"
);
```

Every vertex and edge in the graph carries epoch metadata:

```
Vertex properties:
  created_epoch: 47200        -- when this vertex first appeared
  updated_epoch: 47205        -- last time properties changed
  created_at: '2026-03-14T22:30:00Z'   -- wall clock (for human queries)
  updated_at: '2026-03-14T23:15:00Z'

Edge properties:
  created_epoch: 47201
  valid_from: '2026-03-14T22:31:00Z'   -- when this relationship began
  valid_to: NULL                        -- NULL = still active
  confidence: 0.85
```

### Why Both Epoch and Datetime?

- **Epoch** gives causal ordering: "did this NEAR relationship exist before or after that news event?" — even if wall clocks were slightly off between collectors
- **Datetime** gives human-queryable time windows: "show me everything from the last 6 hours"
- **Epoch** enables efficient graph_accel invalidation: the accelerator already uses a generation counter — epochs align naturally with generations
- **Datetime** enables TTL-based cleanup: "expire observations older than 24h"

## Vertex Lifecycle

### Entities (things that persist and move)

Entities like aircraft, vessels, and satellites have **identity continuity** — the same aircraft exists across many observations. The vertex persists; its position properties update.

```
Epoch 47200: CREATE (e:Entity {entity_id: 'adsb:a88467', lat: 59.0, lng: -141.1, ...})
Epoch 47201: UPDATE e SET lat = 59.1, lng = -141.3, updated_epoch = 47201
Epoch 47202: UPDATE e SET lat = 59.2, lng: -141.5, updated_epoch = 47202
```

Current position is always on the vertex. But we also need the trail.

### Observations (position history)

Each position update creates an **Observation vertex** linked to the Entity:

```cypher
CREATE (o:Observation {
  entity_id: 'adsb:a88467',
  epoch: 47201,
  timestamp: '2026-03-14T22:31:00Z',
  lat: 59.1, lng: -141.3, alt: 10972,
  heading: 300.66, speed_knots: 382.4
})

MATCH (e:Entity {entity_id: 'adsb:a88467'}), (o:Observation {epoch: 47201, entity_id: 'adsb:a88467'})
CREATE (e)-[:OBSERVED {epoch: 47201, timestamp: '2026-03-14T22:31:00Z'}]->(o)
```

This gives us:
- **Current state:** read Entity vertex directly (fast, no traversal)
- **Historical trail:** traverse Entity → OBSERVED → Observation chain (ordered by epoch)
- **Time window replay:** filter Observations by timestamp range

### Events (things that happen once)

Earthquakes, news articles, GDELT events — these are created once, never updated. They carry their epoch and timestamp, and they age out.

```cypher
CREATE (ev:Event {
  event_id: 'usgs:us7000abc1',
  event_type: 'earthquake',
  epoch: 47203,
  timestamp: '2026-03-14T22:30:00Z',
  lat: 38.77, lng: -122.42,
  magnitude: 4.2
})
```

### Relationships (windows of validity)

A NEAR relationship between two entities has a **validity window** — it starts when they enter proximity and ends when they leave:

```cypher
CREATE (e1)-[:NEAR {
  created_epoch: 47201,
  valid_from: '2026-03-14T22:31:00Z',
  valid_to: NULL,                      -- still active
  distance_km: 12.3,
  confidence: 0.95
}]->(e2)
```

When the entities separate:
```cypher
MATCH (e1)-[r:NEAR]->(e2) WHERE r.valid_to IS NULL AND distance(e1, e2) > threshold
SET r.valid_to = '2026-03-14T23:45:00Z', r.ended_epoch = 47250
```

## Time Window Queries

### "What was happening near the Strait of Hormuz between 06:00 and 12:00 UTC?"

```cypher
-- Via graph_accel (sub-millisecond):
SELECT * FROM graph_accel_neighborhood('loc:strait-of-hormuz', 2, 'both', NULL);
-- Then filter results by timestamp range in SQL wrapper

-- Or via Cypher for full temporal precision:
SELECT * FROM cypher('osint', $$
  MATCH (loc:Location {name: 'Strait of Hormuz'})<-[obs:OBSERVED_AT]-(e:Entity)
  WHERE obs.timestamp >= '2026-03-14T06:00:00Z'
    AND obs.timestamp <= '2026-03-14T12:00:00Z'
  RETURN e.entity_type, e.name, obs.timestamp, e.properties
  ORDER BY obs.timestamp
$$) AS (etype agtype, name agtype, ts agtype, props agtype);
```

### "Show me the state of the graph at epoch 47200"

```cypher
-- Entities that existed at epoch 47200
SELECT * FROM cypher('osint', $$
  MATCH (e:Entity)
  WHERE e.created_epoch <= 47200
  RETURN e.entity_id, e.entity_type, e.name
$$) AS (id agtype, etype agtype, name agtype);

-- Relationships active at epoch 47200
SELECT * FROM cypher('osint', $$
  MATCH (a)-[r]->(b)
  WHERE r.created_epoch <= 47200
    AND (r.ended_epoch IS NULL OR r.ended_epoch > 47200)
  RETURN a.entity_id, type(r), b.entity_id, r.confidence
$$) AS (from_id agtype, rel agtype, to_id agtype, conf agtype);
```

### "What changed between epoch 47200 and 47210?"

```cypher
-- New entities
MATCH (e:Entity) WHERE e.created_epoch > 47200 AND e.created_epoch <= 47210
RETURN 'new_entity', e.entity_id, e.entity_type

UNION ALL

-- New relationships
MATCH ()-[r]->() WHERE r.created_epoch > 47200 AND r.created_epoch <= 47210
RETURN 'new_edge', startNode(r).entity_id, type(r)

UNION ALL

-- Ended relationships
MATCH ()-[r]->() WHERE r.ended_epoch > 47200 AND r.ended_epoch <= 47210
RETURN 'ended_edge', startNode(r).entity_id, type(r)
```

## TTL & Cleanup

Not everything lives forever. Different data types have different retention:

```sql
-- Relational config
CREATE TABLE retention_policy (
    vertex_label TEXT PRIMARY KEY,
    ttl_hours INT NOT NULL,
    keep_strategy TEXT CHECK (keep_strategy IN ('delete', 'archive', 'summarize'))
);

INSERT INTO retention_policy VALUES
    ('Observation',  24,  'delete'),      -- position history: 24 hours
    ('Event',        168, 'archive'),     -- events: 7 days, then archive
    ('Entity',       NULL, NULL),         -- entities: never auto-delete
    ('Location',     NULL, NULL),         -- locations: never auto-delete
    ('Report',       NULL, NULL);         -- reports: never auto-delete
```

### Cleanup job (runs hourly)

```sql
-- Delete expired Observations and their edges
SELECT * FROM cypher('osint', $$
  MATCH (o:Observation)
  WHERE o.timestamp < now() - interval '24 hours'
  DETACH DELETE o
$$) AS (result agtype);

-- Archive expired Events (move to relational archive table, remove from graph)
INSERT INTO event_archive (event_id, event_type, timestamp, properties, archived_at)
SELECT event_id, event_type, timestamp, properties, now()
FROM cypher('osint', $$
  MATCH (ev:Event)
  WHERE ev.timestamp < now() - interval '7 days'
  RETURN ev.event_id, ev.event_type, ev.timestamp, ev.properties
$$) AS (eid agtype, etype agtype, ts agtype, props agtype);

-- Then delete from graph
SELECT * FROM cypher('osint', $$
  MATCH (ev:Event)
  WHERE ev.timestamp < now() - interval '7 days'
  DETACH DELETE ev
$$) AS (result agtype);

-- Invalidate accelerator after cleanup
SELECT graph_accel_invalidate('osint');
```

### Entity Staleness

Entities that haven't been observed recently get marked stale, not deleted:

```cypher
MATCH (e:Entity)
WHERE e.updated_at < now() - interval '6 hours'
SET e.stale = true
```

Stale entities render differently on the map (dimmed, no interpolation). If still stale after 48 hours, they can be archived.

## Analysis Model

### What is an "Analysis"?

An analysis is a graph operation that:
1. Takes a **scope** (time window + spatial bounds + entity types)
2. Traverses the graph within that scope
3. Produces **findings** (new edges, annotations, or a report)
4. Records its own provenance (who ran it, when, what scope, what epoch)

### Analysis as a Graph Vertex

```cypher
CREATE (a:Analysis {
  analysis_id: 'analysis:uuid',
  type: 'proximity_scan',           -- or 'correlation', 'anomaly', 'report'
  epoch: 47250,
  timestamp: '2026-03-14T23:00:00Z',
  scope: {
    bbox: [west, south, east, north],
    time_from: '2026-03-14T17:00:00Z',
    time_to: '2026-03-14T23:00:00Z',
    entity_types: ['vessel', 'aircraft']
  },
  created_by: 'claude',             -- or 'user', 'system'
  summary: 'Identified 3 vessel clusters near Strait of Hormuz...'
})
```

Findings link back to the analysis:

```cypher
-- Analysis discovered a correlation between events
MATCH (a:Analysis {analysis_id: 'analysis:uuid'})
MATCH (ev1:Event {event_id: 'gdelt-12345'})
MATCH (ev2:Event {event_id: 'rss:sha256:abcdef'})
CREATE (ev1)-[:RELATED_TO {
  created_epoch: 47250,
  confidence: 0.7,
  correlation_type: 'spatial_temporal',
  discovered_by: a.analysis_id
}]->(ev2)
CREATE (a)-[:PRODUCED]->(ev1)
CREATE (a)-[:PRODUCED]->(ev2)
```

### Analysis Types

#### 1. Proximity Scan (automated, runs every epoch)

For all entities updated this epoch, check if any are newly NEAR each other:

```python
async def proximity_scan(epoch: int, threshold_km: float = 50.0):
    """Create/close NEAR edges based on entity positions."""
    # Get all entities updated this epoch
    updated = await graph.query("""
        MATCH (e:Entity) WHERE e.updated_epoch = $epoch
        RETURN e.entity_id, e.lat, e.lng, e.entity_type
    """, epoch=epoch)

    for entity in updated:
        # Find nearby entities via PostGIS
        nearby = await spatial_query(entity.lat, entity.lng, threshold_km)
        for neighbor in nearby:
            # Create NEAR edge if not already active
            await graph.merge_edge(entity.id, neighbor.id, 'NEAR', {
                'distance_km': distance(entity, neighbor),
                'created_epoch': epoch,
                'valid_from': now(),
                'confidence': 0.95
            })

    # Close NEAR edges where entities have separated
    await graph.query("""
        MATCH (a:Entity)-[r:NEAR]->(b:Entity)
        WHERE r.valid_to IS NULL
        AND distance(a, b) > $threshold
        SET r.valid_to = now(), r.ended_epoch = $epoch
    """, threshold=threshold_km, epoch=epoch)
```

#### 2. Correlation Detection (automated, runs every N epochs)

Look for events that are close in space AND time but from different sources:

```python
async def correlation_scan(time_window_hours: int = 6, distance_km: float = 100):
    """Find events from different sources that are spatially and temporally proximate."""
    correlated = await graph.query("""
        MATCH (e1:Event), (e2:Event)
        WHERE e1.source <> e2.source
          AND abs(e1.timestamp - e2.timestamp) < interval '$hours hours'
          AND distance(e1, e2) < $km
          AND NOT (e1)-[:RELATED_TO]-(e2)
        RETURN e1, e2
    """, hours=time_window_hours, km=distance_km)

    for e1, e2 in correlated:
        confidence = compute_correlation_confidence(e1, e2)
        await graph.create_edge(e1, e2, 'RELATED_TO', {
            'correlation_type': 'spatial_temporal',
            'confidence': confidence,
            'created_epoch': current_epoch()
        })
```

#### 3. Claude Analysis (on-demand, user or MCP triggered)

Claude reads the graph via MCP tools, reasons about connections, produces a report:

```python
# MCP tool: analyze_region
async def analyze_region(bbox, time_range, focus_types=None):
    """Claude calls this to analyze activity in a region."""
    # 1. Query entities and events in scope
    entities = await graph.query_bbox(bbox, time_range, focus_types)

    # 2. Get relationship context via graph_accel
    for entity in key_entities:
        neighborhood = await graph.neighborhood(entity.id, depth=2, min_confidence=0.5)
        # ... Claude reasons about the neighborhood

    # 3. Produce findings
    report = Report(
        title="Activity analysis: Strait of Hormuz, last 6 hours",
        body="...",  # Claude's analysis
        entity_refs=[e.id for e in key_entities],
        epoch=current_epoch()
    )

    # 4. Write report vertex + edges into graph
    await graph.create_vertex('Report', report.to_dict())
    for ref in report.entity_refs:
        await graph.create_edge(report.id, ref, 'REFERENCES', {
            'created_epoch': current_epoch()
        })

    return report
```

#### 4. Anomaly Detection (automated, pattern-based)

Watch for unusual patterns: entity appearing far from last known position, sudden cluster formation, entity going silent.

```python
async def anomaly_scan(epoch: int):
    """Flag entities exhibiting unusual behavior."""
    # Position jump: entity moved impossibly far between observations
    jumps = await graph.query("""
        MATCH (e:Entity)-[:OBSERVED]->(o1:Observation),
              (e)-[:OBSERVED]->(o2:Observation)
        WHERE o2.epoch = $epoch AND o1.epoch = $epoch - 1
          AND distance(o1, o2) > max_plausible_distance(e.entity_type, interval)
        RETURN e, o1, o2
    """, epoch=epoch)

    for entity, prev, curr in jumps:
        await graph.create_vertex('Event', {
            'event_type': 'anomaly',
            'anomaly_type': 'position_jump',
            'entity_id': entity.id,
            'epoch': epoch,
            'description': f'{entity.name} jumped {distance}km in {interval}s'
        })
```

## Epoch Advancement

When does the epoch advance?

```python
class EpochManager:
    """Advances epoch on material graph changes."""

    async def advance(self, collector: str, changes: ChangeSet) -> int:
        """Called by collectors after ingest. Returns new epoch."""
        if not changes.is_material():
            return self.current_epoch  # no-op for trivial updates

        new_epoch = await self.db.execute("""
            INSERT INTO epochs (collector, entity_count, edge_count)
            VALUES ($1, $2, $3)
            RETURNING epoch
        """, collector, changes.entity_count, changes.edge_count)

        # Invalidate graph_accel cache
        await self.db.execute("SELECT graph_accel_invalidate('osint')")

        return new_epoch

    def is_material(self, changes: ChangeSet) -> bool:
        """Position updates alone don't advance epoch. New entities/edges do."""
        return (changes.new_entities > 0
                or changes.new_edges > 0
                or changes.closed_edges > 0
                or changes.new_events > 0)
```

**Key design decision:** Position updates on existing entities do NOT advance the epoch. Only structural changes (new vertices, new edges, closed edges) advance it. This prevents epoch inflation from the fast-tier 60-second position polling while ensuring every meaningful graph mutation is ordered.

## Epoch Window Comparisons

An **epoch window** is a named selection of time — not necessarily contiguous. You can select arbitrary clusters of epochs and cross-cut them against entity types, spatial bounds, or relationship types.

### Window as a Query Primitive

```python
@dataclass
class EpochWindow:
    name: str                              # "pre-exercise", "during-exercise"
    ranges: list[tuple[int, int]]          # [(47200, 47210), (47250, 47255)]
    # Can also express as datetime ranges, resolved to epochs at query time
    time_ranges: list[tuple[datetime, datetime]] | None = None

    def to_epoch_filter(self) -> str:
        """Generate Cypher WHERE clause."""
        clauses = [f"(n.epoch >= {lo} AND n.epoch <= {hi})" for lo, hi in self.ranges]
        return " OR ".join(clauses)
```

### Multi-Window Comparison

Compare entity activity across two or more windows:

```python
async def compare_windows(
    windows: list[EpochWindow],
    entity_types: list[str] | None = None,
    bbox: BBox | None = None,
    edge_types: list[str] | None = None,
) -> ComparisonResult:
    """Cross-cut multiple epoch windows against entity/edge filters."""

    results = {}
    for window in windows:
        # Entities active in this window
        entities = await graph.query(f"""
            MATCH (e:Entity)-[:OBSERVED]->(o:Observation)
            WHERE ({window.to_epoch_filter('o')})
            {'AND e.entity_type IN $types' if entity_types else ''}
            RETURN DISTINCT e.entity_id, e.entity_type, e.name,
                   count(o) AS observation_count,
                   min(o.timestamp) AS first_seen,
                   max(o.timestamp) AS last_seen
        """, types=entity_types)

        # Relationships active in this window
        edges = await graph.query(f"""
            MATCH (a)-[r]->(b)
            WHERE ({window.to_epoch_filter('r')})
            {'AND type(r) IN $edge_types' if edge_types else ''}
            RETURN a.entity_id, type(r), b.entity_id, r.confidence
        """, edge_types=edge_types)

        # Events in this window
        events = await graph.query(f"""
            MATCH (ev:Event)
            WHERE ({window.to_epoch_filter('ev')})
            RETURN ev.event_id, ev.event_type, ev.timestamp, ev.lat, ev.lng
        """)

        results[window.name] = WindowSnapshot(
            entities=entities,
            edges=edges,
            events=events
        )

    return ComparisonResult(
        windows=results,
        diff=compute_diff(results),   # what appeared/disappeared between windows
        overlap=compute_overlap(results)  # what persisted across all windows
    )
```

### Example: Before/During/After a Naval Exercise

```python
windows = [
    EpochWindow("before", time_ranges=[
        (datetime(2026,3,10), datetime(2026,3,12))
    ]),
    EpochWindow("during", time_ranges=[
        (datetime(2026,3,12), datetime(2026,3,14))
    ]),
    EpochWindow("after", time_ranges=[
        (datetime(2026,3,14), datetime(2026,3,16))
    ]),
]

result = await compare_windows(
    windows,
    entity_types=["vessel", "aircraft"],
    bbox=BBox(west=25, south=33, east=37, north=42),  # Eastern Mediterranean
)

# result.diff tells you:
#   - Which vessels appeared during the exercise that weren't there before
#   - Which aircraft showed up only during the exercise window
#   - Which NEAR relationships formed during the exercise
#   - What disappeared after the exercise ended
```

### Example: Non-Contiguous Comparison

Compare activity during multiple separate incidents:

```python
windows = [
    EpochWindow("suez-incident", time_ranges=[
        (datetime(2026,1,15,6,0), datetime(2026,1,15,18,0))
    ]),
    EpochWindow("hormuz-incident", time_ranges=[
        (datetime(2026,2,22,3,0), datetime(2026,2,22,15,0))
    ]),
    EpochWindow("bab-el-mandeb-incident", time_ranges=[
        (datetime(2026,3,8,9,0), datetime(2026,3,8,21,0))
    ]),
]

result = await compare_windows(
    windows,
    entity_types=["vessel"],
    edge_types=["NEAR"],
)

# result.overlap tells you:
#   - Were any of the same vessels present at all three chokepoints?
#   - Did the same vessel-to-vessel NEAR relationships recur?
#   - Cross-incident pattern: same actors showing up repeatedly
```

### Diff Structure

```python
@dataclass
class WindowDiff:
    appeared: dict[str, set[str]]     # window_name → entity_ids that first appear
    disappeared: dict[str, set[str]]  # window_name → entity_ids that were in previous but not this
    persisted: set[str]               # entity_ids present in ALL windows
    edges_formed: dict[str, list]     # window_name → new edges
    edges_closed: dict[str, list]     # window_name → edges that ended
    event_density: dict[str, int]     # window_name → event count (is activity increasing?)
```

### MCP Tools for Window Analysis

```
create_window(name, time_ranges)            — define a named epoch window
compare_windows(window_names, filters)      — cross-cut comparison
window_diff(window_a, window_b, filters)    — what changed between two windows
window_timeline(windows, entity_id)         — trace one entity across windows
```

Claude can use these to build temporal narratives: "The same three cargo vessels appeared at Suez on Jan 15, Hormuz on Feb 22, and Bab-el-Mandeb on Mar 8. Each time, they were NEAR the same tanker. Here's the pattern..."

### Epoch Delta & Vectors

Selecting two epochs (or two windows) gives you the **delta** — not just what changed, but the vectors of change:

```python
@dataclass
class EntityVector:
    entity_id: str
    entity_type: str
    # Position delta
    position_a: tuple[float, float]       # (lat, lng) at epoch A
    position_b: tuple[float, float]       # (lat, lng) at epoch B
    displacement_km: float                 # great-circle distance moved
    bearing: float                         # compass heading of movement
    speed_avg_knots: float                 # displacement / time between epochs
    # Relationship delta
    edges_gained: list[EdgeDelta]          # new relationships since epoch A
    edges_lost: list[EdgeDelta]            # relationships that closed since epoch A
    # Degree delta
    degree_a: int                          # total connections at epoch A
    degree_b: int                          # total connections at epoch B
    degree_delta: int                      # positive = gaining connections

@dataclass
class EpochDelta:
    epoch_a: int
    epoch_b: int
    time_elapsed: timedelta
    entity_vectors: list[EntityVector]     # per-entity movement + relationship changes
    new_entities: list[str]                # appeared between A and B
    gone_entities: list[str]               # present at A, absent at B
    new_events: list[str]                  # events that occurred between A and B
    edge_formation_rate: float             # new edges per hour
    edge_dissolution_rate: float           # closed edges per hour
    event_density: float                   # events per hour (is it accelerating?)
```

```python
async def epoch_delta(epoch_a: int, epoch_b: int,
                      entity_types: list[str] | None = None,
                      bbox: BBox | None = None) -> EpochDelta:
    """Compute the full delta between two graph states."""

    # Entity position vectors
    vectors = await graph.query("""
        MATCH (e:Entity)
        WHERE e.created_epoch <= $b
        OPTIONAL MATCH (e)-[:OBSERVED]->(oa:Observation)
          WHERE oa.epoch <= $a ORDER BY oa.epoch DESC LIMIT 1
        OPTIONAL MATCH (e)-[:OBSERVED]->(ob:Observation)
          WHERE ob.epoch <= $b ORDER BY ob.epoch DESC LIMIT 1
        RETURN e.entity_id, e.entity_type,
               oa.lat, oa.lng, oa.epoch,
               ob.lat, ob.lng, ob.epoch
    """, a=epoch_a, b=epoch_b)

    # For each entity, compute displacement and bearing
    entity_vectors = []
    for v in vectors:
        if v.oa_lat and v.ob_lat:
            disp = haversine(v.oa_lat, v.oa_lng, v.ob_lat, v.ob_lng)
            bearing = initial_bearing(v.oa_lat, v.oa_lng, v.ob_lat, v.ob_lng)
            elapsed = epoch_to_time(v.ob_epoch) - epoch_to_time(v.oa_epoch)
            speed = (disp / elapsed.total_seconds()) * 1943.844  # m/s to knots
        # ... build EntityVector

    # Relationship deltas
    gained = await graph.query("""
        MATCH ()-[r]->()
        WHERE r.created_epoch > $a AND r.created_epoch <= $b
        RETURN startNode(r).entity_id, type(r), endNode(r).entity_id
    """, a=epoch_a, b=epoch_b)

    lost = await graph.query("""
        MATCH ()-[r]->()
        WHERE r.ended_epoch > $a AND r.ended_epoch <= $b
        RETURN startNode(r).entity_id, type(r), endNode(r).entity_id
    """, a=epoch_a, b=epoch_b)

    return EpochDelta(
        epoch_a=epoch_a, epoch_b=epoch_b,
        entity_vectors=entity_vectors,
        # ... new_entities, gone_entities, rates
    )
```

#### Rendering Vectors on the Map

Entity vectors render as arrows from position_a to position_b:
- **Arrow length** proportional to displacement
- **Arrow color** by entity type (cyan=aircraft, blue=vessel, etc.)
- **Arrow thickness** by speed (faster = bolder)
- **Ghosted position** at epoch A (faded icon), solid position at epoch B
- **Relationship formation** rendered as green edges appearing
- **Relationship dissolution** rendered as red edges fading

This gives you an at-a-glance view of "which way is everything moving, and what new connections formed" — like a weather map for geopolitical activity.

#### MCP Tools

```
epoch_delta(epoch_a, epoch_b, filters)     — full delta with vectors
entity_trajectory(entity_id, epoch_range)  — single entity's path over time
formation_rate(edge_type, epoch_range)     — how fast are relationships forming?
```

Claude can use `epoch_delta` to narrate: "Between 06:00 and 12:00, 4 cargo vessels moved SE through Hormuz at 12-15 knots. During the same window, 3 military aircraft appeared in the region and 2 new NEAR relationships formed between the cargo vessels and a destroyer..."

### Multi-Window Pattern Matching

When you select N epoch windows, you're not just comparing pairs — you're building a **behavioral signature** that can be matched against other entities or other time periods.

#### Trajectory Fingerprinting

An entity's observations across a window form a trajectory — a sequence of (position, timestamp) tuples. Compare trajectories across windows to find patterns:

```python
@dataclass
class Trajectory:
    entity_id: str
    window: EpochWindow
    points: list[tuple[float, float, datetime]]  # (lat, lng, timestamp)
    total_distance_km: float
    avg_speed_knots: float
    bearing_changes: list[float]                   # heading deltas at each step
    dwell_points: list[DwellPoint]                 # locations where speed ≈ 0

@dataclass
class DwellPoint:
    lat: float
    lng: float
    arrived: datetime
    departed: datetime
    duration: timedelta

async def extract_trajectory(entity_id: str, window: EpochWindow) -> Trajectory:
    """Build trajectory from observations within a window."""
    observations = await graph.query("""
        MATCH (e:Entity {entity_id: $eid})-[:OBSERVED]->(o:Observation)
        WHERE o.timestamp >= $t1 AND o.timestamp <= $t2
        RETURN o.lat, o.lng, o.timestamp
        ORDER BY o.timestamp
    """, eid=entity_id, t1=window.start, t2=window.end)

    points = [(o.lat, o.lng, o.timestamp) for o in observations]
    dwell_points = detect_dwells(points, speed_threshold=0.5)  # knots
    bearing_changes = compute_bearing_deltas(points)

    return Trajectory(
        entity_id=entity_id,
        window=window,
        points=points,
        total_distance_km=sum_segments(points),
        avg_speed_knots=avg_speed(points),
        bearing_changes=bearing_changes,
        dwell_points=dwell_points
    )
```

#### "What kind of flight path best matches this aircraft?"

Compare a current aircraft's trajectory against known patterns:

```python
async def classify_trajectory(
    entity_id: str,
    current_window: EpochWindow,
    reference_windows: list[EpochWindow],  # historical windows to compare against
    candidate_entity_types: list[str] | None = None,
) -> list[TrajectoryMatch]:
    """Find historical trajectories that best match the current one."""

    current = await extract_trajectory(entity_id, current_window)

    # Build reference library from historical windows
    reference_trajectories = []
    for window in reference_windows:
        entities = await graph.query("""
            MATCH (e:Entity)-[:OBSERVED]->(o:Observation)
            WHERE o.timestamp >= $t1 AND o.timestamp <= $t2
            RETURN DISTINCT e.entity_id, e.entity_type
        """, t1=window.start, t2=window.end)

        for entity in entities:
            if candidate_entity_types and entity.type not in candidate_entity_types:
                continue
            traj = await extract_trajectory(entity.id, window)
            if len(traj.points) >= 3:  # minimum for meaningful comparison
                reference_trajectories.append(traj)

    # Score each reference trajectory against current
    matches = []
    for ref in reference_trajectories:
        score = trajectory_similarity(current, ref)
        matches.append(TrajectoryMatch(
            reference_entity=ref.entity_id,
            reference_window=ref.window.name,
            similarity=score,
            matching_features=explain_match(current, ref)
        ))

    return sorted(matches, key=lambda m: m.similarity, reverse=True)
```

#### Similarity Metrics

Trajectory comparison uses multiple features, weighted by relevance:

```python
def trajectory_similarity(a: Trajectory, b: Trajectory) -> float:
    """Score 0.0-1.0 for how similar two trajectories are."""
    scores = {
        # Shape similarity: do they follow the same path geometry?
        'shape': frechet_distance_normalized(a.points, b.points),

        # Speed profile: similar acceleration/deceleration patterns?
        'speed': dtw_distance(a.speed_profile, b.speed_profile),

        # Bearing pattern: similar turns at similar points?
        'bearing': dtw_distance(a.bearing_changes, b.bearing_changes),

        # Dwell pattern: do they stop at similar locations for similar durations?
        'dwell': dwell_pattern_similarity(a.dwell_points, b.dwell_points),

        # Total distance: similar trip length?
        'distance': 1.0 - abs(a.total_distance_km - b.total_distance_km)
                         / max(a.total_distance_km, b.total_distance_km, 1),
    }

    weights = {'shape': 0.35, 'speed': 0.2, 'bearing': 0.2, 'dwell': 0.15, 'distance': 0.1}
    return sum(scores[k] * weights[k] for k in weights)
```

#### Use Cases

**"Is this a patrol route?"** — Compare current military aircraft trajectory against its own past trajectories across N previous windows. High self-similarity = repeating patrol. Low = unusual deviation.

**"What vessel typically takes this path?"** — Given a trajectory through a strait, find which vessels historically follow the same route. Reveals regular shipping lanes vs unusual transits.

**"This flight path looks like reconnaissance"** — Compare an unidentified aircraft's trajectory against a library of known recon patterns (racetrack orbits, figure-8s, border paralleling). The bearing_changes feature alone distinguishes loiter patterns from transit flights.

**"Has this happened before?"** — Select the current anomalous window, compare against all historical windows of similar duration. Find the closest match. Claude narrates: "The current activity pattern in the Eastern Med most closely resembles the pattern observed during [exercise name] on [date], with 0.82 similarity."

#### MCP Tools for Pattern Analysis

```
extract_trajectory(entity_id, window)                    — get trajectory data
classify_trajectory(entity_id, window, reference_windows) — pattern matching
compare_trajectories(entity_a, window_a, entity_b, window_b) — pairwise comparison
find_similar_windows(current_window, lookback_days)       — "has this happened before?"
```

### MCP Server as Analysis Engine

The MCP server is the compute layer between Claude and the graph. Claude asks questions in natural language; MCP tools do the heavy math and return clean textual summaries. Claude never sees raw coordinate arrays or graph traversal results — it gets pre-digested intelligence.

```python
# MCP tool implementation pattern
@mcp_tool("classify_flight")
async def classify_flight(callsign: str, hours_back: int = 6) -> str:
    """Classify a flight's behavior pattern. Returns text summary for Claude."""
    entity = await graph.find_entity(f"callsign:{callsign}")
    if not entity:
        return f"No active flight found with callsign {callsign}."

    window = EpochWindow("current", time_ranges=[
        (datetime.utcnow() - timedelta(hours=hours_back), datetime.utcnow())
    ])
    trajectory = await extract_trajectory(entity.id, window)

    # Compare against reference patterns
    matches = await classify_trajectory(entity.id, window,
        reference_windows=last_n_windows(days=30),
        candidate_entity_types=["aircraft"])

    # Build text summary for Claude
    lines = [
        f"## Flight {callsign} — Trajectory Analysis",
        f"",
        f"**Duration:** {trajectory.duration}",
        f"**Distance:** {trajectory.total_distance_km:.0f} km",
        f"**Avg speed:** {trajectory.avg_speed_knots:.0f} knots",
        f"**Dwell points:** {len(trajectory.dwell_points)}",
    ]

    if trajectory.dwell_points:
        lines.append(f"**Longest dwell:** {trajectory.dwell_points[0].duration} "
                     f"at ({trajectory.dwell_points[0].lat:.2f}, "
                     f"{trajectory.dwell_points[0].lng:.2f})")

    bearing_variance = statistics.stdev(trajectory.bearing_changes) if len(trajectory.bearing_changes) > 1 else 0
    if bearing_variance > 30:
        lines.append(f"**Flight pattern:** Loiter/orbit (high bearing variance: {bearing_variance:.0f}°)")
    elif bearing_variance < 5:
        lines.append(f"**Flight pattern:** Direct transit (bearing variance: {bearing_variance:.0f}°)")
    else:
        lines.append(f"**Flight pattern:** Mixed (bearing variance: {bearing_variance:.0f}°)")

    if matches:
        lines.append(f"")
        lines.append(f"### Best Historical Matches")
        for m in matches[:3]:
            lines.append(f"- **{m.reference_entity}** during {m.reference_window}: "
                        f"{m.similarity:.0%} similar ({', '.join(m.matching_features)})")

    return "\n".join(lines)
```

Claude receives something like:

```
## Flight RRR7142 — Trajectory Analysis

**Duration:** 4h 23m
**Distance:** 312 km
**Avg speed:** 185 knots
**Dwell points:** 2
**Longest dwell:** 47 minutes at (34.82, 33.15)
**Flight pattern:** Loiter/orbit (high bearing variance: 42°)

### Best Historical Matches
- **RRR7108** during patrol-feb-22: 87% similar (shape, bearing pattern, dwell locations)
- **RRR7155** during patrol-jan-15: 79% similar (shape, speed profile)
- **USN-P8A-201** during exercise-mar-01: 71% similar (bearing pattern, dwell duration)
```

Claude can then say: "This looks like a standard patrol pattern — it matches 3 previous flights from the same unit with 79-87% similarity, including similar dwell points near the coast."

### Event Criticality & Emerging Situations

Events carry a **criticality score** that evolves over time based on graph context, not just the initial source data.

#### Scoring Model

```python
@dataclass
class EventScore:
    event_id: str
    base_score: float          # from source (e.g., earthquake magnitude, news risk_score)
    graph_score: float         # from graph context (computed)
    velocity: float            # rate of score change (is it accelerating?)
    trend: str                 # 'rising', 'stable', 'declining'
    components: dict           # what's driving the score

async def score_event(event_id: str) -> EventScore:
    """Compute event criticality from graph context."""

    event = await graph.get_vertex(event_id)
    base = event.properties.get('magnitude', 0) or event.properties.get('risk_score', 0)

    # Graph-derived signals
    neighborhood = await graph.neighborhood(event_id, depth=2)

    components = {
        # How many other events are correlated?
        'correlation_count': len([n for n in neighborhood if n.label == 'Event']),

        # How many entities are involved?
        'entity_involvement': len([n for n in neighborhood if n.label == 'Entity']),

        # Are high-value entities nearby? (military, carriers, tracked)
        'high_value_proximity': sum(1 for n in neighborhood
            if n.label == 'Entity'
            and n.properties.get('classification') in ('military', 'tracked', 'carrier')),

        # News density: how many news articles reference this area recently?
        'news_density': await count_nearby_events(event, type='news_article', hours=6),

        # Is this in a historically active region?
        'historical_activity': await region_activity_baseline(event.lat, event.lng),
    }

    graph_score = weighted_sum(components, weights={
        'correlation_count': 0.25,
        'entity_involvement': 0.20,
        'high_value_proximity': 0.25,
        'news_density': 0.15,
        'historical_activity': 0.15,
    })

    # Velocity: compare score to previous epochs
    previous_scores = await get_score_history(event_id, lookback_epochs=10)
    velocity = linear_regression_slope(previous_scores) if len(previous_scores) >= 3 else 0.0
    trend = 'rising' if velocity > 0.1 else 'declining' if velocity < -0.1 else 'stable'

    return EventScore(
        event_id=event_id,
        base_score=base,
        graph_score=graph_score,
        velocity=velocity,
        trend=trend,
        components=components
    )
```

#### Emerging Situation Detection

An **emerging situation** is a cluster of events whose aggregate score is accelerating:

```python
async def detect_emerging(
    min_velocity: float = 0.2,
    min_cluster_size: int = 3,
    hours_back: int = 6,
) -> list[EmergingSituation]:
    """Find clusters of events with rising criticality."""

    recent_events = await graph.query("""
        MATCH (ev:Event)
        WHERE ev.timestamp > now() - interval '$hours hours'
        RETURN ev
        ORDER BY ev.timestamp DESC
    """, hours=hours_back)

    # Spatial clustering (DBSCAN-style)
    clusters = spatial_cluster(recent_events, eps_km=100, min_samples=min_cluster_size)

    emerging = []
    for cluster in clusters:
        scores = [await score_event(ev.event_id) for ev in cluster.events]
        avg_velocity = statistics.mean(s.velocity for s in scores)
        max_score = max(s.graph_score for s in scores)

        if avg_velocity >= min_velocity:
            # Score is accelerating — this cluster is heating up
            emerging.append(EmergingSituation(
                center=(cluster.centroid_lat, cluster.centroid_lng),
                events=cluster.events,
                scores=scores,
                avg_velocity=avg_velocity,
                max_score=max_score,
                trend='accelerating',
                summary=generate_situation_summary(cluster, scores),
            ))

    return sorted(emerging, key=lambda e: e.avg_velocity, reverse=True)
```

#### Score History (stored per-epoch)

```sql
-- Relational table for score time series
CREATE TABLE event_scores (
    event_id TEXT NOT NULL,
    epoch BIGINT NOT NULL,
    base_score FLOAT,
    graph_score FLOAT,
    velocity FLOAT,
    components JSONB,
    scored_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (event_id, epoch)
);

CREATE INDEX idx_scores_velocity ON event_scores (velocity DESC)
    WHERE velocity > 0.1;  -- fast lookup for "rising" events
```

#### MCP Tool for Claude

```python
@mcp_tool("emerging_situations")
async def emerging_situations(hours_back: int = 6) -> str:
    """Find emerging situations. Returns text summary for Claude."""
    situations = await detect_emerging(hours_back=hours_back)

    if not situations:
        return "No emerging situations detected in the last {hours_back} hours."

    lines = [f"## {len(situations)} Emerging Situation(s) — Last {hours_back}h", ""]

    for i, sit in enumerate(situations, 1):
        lines.append(f"### Situation {i}: {sit.summary}")
        lines.append(f"**Location:** ({sit.center[0]:.2f}, {sit.center[1]:.2f})")
        lines.append(f"**Events:** {len(sit.events)}")
        lines.append(f"**Max criticality:** {sit.max_score:.2f}")
        lines.append(f"**Acceleration:** {sit.avg_velocity:+.2f}/epoch ({sit.trend})")
        lines.append(f"**Key drivers:**")
        # Aggregate the top scoring components across the cluster
        top_drivers = aggregate_components(sit.scores)
        for driver, value in top_drivers[:3]:
            lines.append(f"  - {driver}: {value}")
        lines.append("")

    return "\n".join(lines)
```

Claude sees:

```
## 2 Emerging Situation(s) — Last 6h

### Situation 1: Eastern Mediterranean naval buildup
**Location:** (34.50, 33.20)
**Events:** 7
**Max criticality:** 0.78
**Acceleration:** +0.35/epoch (accelerating)
**Key drivers:**
  - high_value_proximity: 3 military entities
  - correlation_count: 5 correlated events
  - news_density: 12 articles in 6h

### Situation 2: Strait of Hormuz transit cluster
**Location:** (26.55, 56.25)
**Events:** 4
**Max criticality:** 0.52
**Acceleration:** +0.22/epoch (accelerating)
**Key drivers:**
  - entity_involvement: 8 vessels
  - correlation_count: 3 correlated events
  - historical_activity: above 90th percentile
```

### Frontend Integration

The map can render window comparisons as layer overlays:
- **Window A entities** in one color, **Window B entities** in another
- **Overlap** entities highlighted (present in both windows)
- **Timeline scrubber** that snaps to epoch boundaries within a window
- **Side-by-side** mode: split map showing two windows simultaneously

## Graph Size Management

### Estimated Scale (steady state, 24h retention on observations)

| Vertex Type | Estimate | Growth Rate |
|-------------|----------|-------------|
| Entity | ~5,000 (aircraft + vessels + satellites + infra) | Slow (new entities appear gradually) |
| Observation | ~120,000 (5000 entities × 24 obs/day) | 5,000/hour, pruned at 24h |
| Event | ~2,000 (news + earthquakes + GDELT + fires) | ~200/hour, pruned at 7 days |
| Location | ~10,000 (airports + ports + cities + data centers) | Near-zero (reference data) |
| Analysis | ~50/day | Kept indefinitely |
| Report | ~10/day | Kept indefinitely |

| Edge Type | Estimate | Notes |
|-----------|----------|-------|
| OBSERVED | ~120,000 | 1:1 with Observations, pruned together |
| NEAR | ~500-2,000 active | Opened/closed as entities move |
| RELATED_TO | ~500/day | Correlation scan results |
| REPORTED_BY | 1:1 with Events | |
| OBSERVED_AT | ~5,000 active | Entity → Location |

**Total steady state:** ~140,000 vertices, ~130,000 edges. Well within graph_accel's tested range (benchmarked to 5M nodes / 50M edges).

### Archive Strategy

After TTL expiry:
1. **Observations** → deleted (position history is transient)
2. **Events** → archived to relational `event_archive` table (queryable but not in graph)
3. **NEAR edges** → closed edges deleted after 48h (the fact they existed is in the Analysis/Report that referenced them)
4. **Analysis/Report** vertices → kept indefinitely (they ARE the institutional memory)

The graph stays lean. The archive stays complete. Analysis reports reference entities by ID, so even after an entity leaves the graph, the report still makes sense — it's a snapshot-in-time conclusion, not a live query.
