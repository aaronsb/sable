# Sable

Open source intelligence collection and analysis platform. Aggregates real-time data from 30+ open sources — flight tracking, maritime AIS, satellite positions, conflict maps, news feeds, radio, traffic cameras, seismic activity — into an interactive map backed by a graph database.

## Status

**Not built yet.** This repo currently contains only architecture documents and a detailed spec in `reference/`. The spec was extracted from an existing tool that does most of this, but does it badly (monolithic, no database, no analysis capability, leaked credentials in git history). Sable is a clean-room rebuild with a real architecture.

## What It Will Do

- Collect OSINT from pluggable source modules (ADS-B flights, AIS maritime, CelesTrak satellites, GDELT events, RSS news, NASA FIRMS fires, USGS earthquakes, and more)
- Store everything in a PostgreSQL graph (Apache AGE) with spatial indexing (PostGIS) and a Rust-based graph accelerator for sub-millisecond multi-hop traversals
- Track entities over time using an epoch-based temporal model — compare arbitrary time windows, compute movement vectors, fingerprint trajectories, detect emerging situations
- Expose an MCP server so Claude can query the graph, draw on the map, run analysis, and post reports
- Optionally federate with a semantic knowledge graph (Kappa) to bridge operational observations with conceptual reasoning

## Architecture

See `reference/06-proposed-architecture.md` for the full design, or `CLAUDE.md` for the summary.

## License

TBD
