# Sable — OSINT Collection & Analysis Platform

## What This Is

Sable is a **clean-room rebuild** of a geopolitical OSINT dashboard. It collects real-time data from 30+ open sources (flight tracking, maritime AIS, conflict maps, news, satellites, radio, cameras, earthquakes, fires) and renders them on an interactive map with analysis capabilities.

The `reference/` directory contains a detailed spec extracted from an existing tool called ShadowBroker. That code is not part of this project — it was used purely as a feature inventory and capability reference. There is no license from the original, so nothing is carried forward except ideas.

## Architecture (read `reference/06-proposed-architecture.md` for full detail)

**Stack:** Python (FastAPI) backend, Next.js frontend, PostgreSQL with three capabilities:

1. **Relational** (application tables) — collectors, credentials, annotations, reports
2. **Apache AGE graph** (OSINT entity model) — entities, events, observations, relationships via openCypher
3. **PostGIS spatial** — extends both layers with geometry indexing and ST_* functions

**Key architectural decisions:**
- **Pluggable collectors** — each data source is a self-contained module implementing a `Collector` protocol, discovered and injected at startup. Missing credentials disable the collector, not crash the app.
- **graph_accel** — a Rust-based PostgreSQL extension (from the Kappa project at `/home/aaron/Projects/ai/knowledge-graph-system/graph-accel/`) that accelerates AGE graph traversals by 36,000x–244,600x via in-memory adjacency lists. This is not optional — multi-hop queries hang without it.
- **Epoch-based temporal model** — monotonically increasing logical clock for causal ordering. Position updates don't advance epochs; only structural changes (new entities, edges, events) do. Enables time-window comparisons, trajectory fingerprinting, and delta/vector analysis between epochs.
- **MCP server** — Claude connects via MCP to query the graph, plot markers/shapes on the map, run analysis (trajectory classification, emerging situation detection, window comparisons), and post reports. The MCP server does the heavy compute and returns clean text summaries.
- **Kappa graph federation** — optional sync to the Kappa knowledge graph system. OSINT events promote to Kappa concepts as supporting evidence, boosting grounding scores with empirical backing. Kappa's semantic traversal discovers connections OSINT can't (e.g., linking geographically distant events via conceptual chains). Bidirectional foreign references bridge the two graphs.

## Reference Documents

Read these before writing code. They are the spec.

| Doc | Contents |
|-----|----------|
| `reference/01-data-sources.md` | All 30+ external data sources with APIs, auth requirements, polling cadence |
| `reference/02-entity-types.md` | Entity taxonomy with property shapes (aircraft, vessels, satellites, events, etc.) |
| `reference/03-map-layers.md` | 48+ visualization layers, icon systems, interaction patterns |
| `reference/04-ui-panels.md` | UI panel inventory (layer controls, entity detail, news feed, radio, markets) |
| `reference/05-architecture-antipatterns.md` | What the original got wrong (monolith, no DB, no pagination, .env secrets, full GeoJSON rebuilds) |
| `reference/06-proposed-architecture.md` | Full architecture: collector protocol, event schema, database schema (relational + AGE + PostGIS), graph_accel config, API design, MCP tools, frontend architecture |
| `reference/07-frontend-architecture.md` | Detailed frontend decomposition: data flow, GeoJSON builders, interpolation math, icon system, click handling, popup data, state management, performance patterns |
| `reference/08-temporal-model.md` | Epoch model, vertex lifecycle, TTL/cleanup, window comparisons, epoch deltas/vectors, trajectory fingerprinting, event criticality scoring, emerging situation detection |
| `reference/09-kappa-fusion.md` | Two-graph fusion: foreign references, semantic grounding from OSINT, reverse traversal, intelligence cycle |

## Related Projects

- **Kappa Knowledge Graph:** `/home/aaron/Projects/ai/knowledge-graph-system/` — semantic knowledge graph with AGE backend. Contains the `graph-accel/` Rust extension (pgrx-based PostgreSQL extension). The same `graph_accel.so` binary works in both projects since they share the same AGE storage model.
- **ShadowBroker (reference only):** `/home/aaron/Projects/app/Shadowbroker/` — the original forked tool used as feature reference. History was flattened and fork severed from `BigBodyCobain/Shadowbroker`. Do not extend this code.

## Design Principles

1. **Modular over monolithic** — one collector per source, one layer component per entity type, routers not god-files
2. **Graph-first data model** — entities and relationships in AGE, not JSON blobs in memory
3. **Temporal integrity** — everything has an epoch and timestamp, nothing is silently overwritten
4. **Analysis as a first-class operation** — not an afterthought bolted onto a dashboard
5. **No edgelord aesthetics** — dark mode with muted palette, professional not theatrical
6. **Real secrets management** — no `.env` files for API tokens in production
7. **Scan before you run** — dependency audit before installing anything from external sources

## Implementation Phasing

These are roughly ordered but not rigid:

- **Phase 0:** Project scaffolding — FastAPI backend, Next.js frontend, PostgreSQL + AGE + PostGIS docker-compose for the database, Makefile
- **Phase 1:** Collector framework — the `Collector` protocol, discovery/registration, scheduler, one working collector (flights via adsb.lol, no auth required) to prove the pipeline
- **Phase 2:** Graph schema — AGE graph initialization, vertex/edge label creation, the `Event` normalization pipeline from collector output to graph vertices
- **Phase 3:** API + basic frontend — REST endpoints for entities/events with pagination, MapLibre map rendering one entity type, WebSocket for real-time push
- **Phase 4:** graph_accel integration — install the Rust extension, configure GUCs, implement `GraphFacade` with accelerated traversal + Cypher fallback
- **Phase 5:** Additional collectors — add sources incrementally (maritime, satellites, news, etc.), each as a pluggable module
- **Phase 6:** Temporal model — epoch management, observation history, TTL cleanup, window queries
- **Phase 7:** Analysis — proximity scan, correlation detection, event scoring, emerging situation detection
- **Phase 8:** MCP server — Claude integration for querying, plotting, analysis, reports
- **Phase 9:** Kappa federation — foreign references, promotion pipeline, semantic grounding bridge
