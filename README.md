# Sable

Open source intelligence collection and analysis platform. Aggregates real-time data from 30+ open sources — flight tracking, maritime AIS, satellite positions, conflict maps, news feeds, radio, traffic cameras, seismic activity — into an interactive map backed by a graph database.

## Status

**Not built yet.** This repo currently contains only architecture documents and a detailed spec in `reference/`. The spec was extracted from an existing tool that does most of this, but does it badly (monolithic, no database, no analysis capability, leaked credentials in git history). Sable is a clean-room rebuild with a real architecture.

## The Idea

Most OSINT tools are dashboards — they show you dots on a map and call it situational awareness. Sable is designed to be something different: a platform where collected data flows into a graph, the graph reveals relationships that aren't obvious from coordinates alone, and an AI can traverse those relationships to build narratives and direct further collection.

The interesting part isn't any single data source. It's what happens when you connect flight tracks to maritime positions to news events to satellite imagery over time, store all of it as a graph with temporal epochs, and then ask "has this pattern happened before?" or "what's accelerating right now?"

And when that operational graph federates with a semantic knowledge graph, you get a system that can bridge observable events ("Russian cargo vessel docks at Tartus") with conceptual chains ("Russian military logistics implies Wagner Group resupply implies communications spike in Eastern Libya") — and trace those inferences back to the actual OSINT events that ground them.

We think this could be an open source alternative to platforms that currently cost seven figures and still can't do the graph traversal part.

## What It Will Do

- **Collect** — pluggable source modules for 30+ open data feeds (ADS-B flights, AIS maritime, CelesTrak satellites, GDELT events, RSS news, NASA FIRMS fires, USGS earthquakes, traffic cameras, radio, and more)
- **Store** — PostgreSQL with Apache AGE (graph), PostGIS (spatial), and a Rust-based graph accelerator for sub-millisecond multi-hop traversals
- **Track** — epoch-based temporal model for comparing arbitrary time windows, computing movement vectors, fingerprinting trajectories, and detecting emerging situations
- **Analyze** — MCP server so Claude can query the graph, draw on the map, classify behavior patterns, and post reports that feed back into the graph
- **Fuse** — optional federation with a semantic knowledge graph to bridge observations with meaning

## Architecture

The `reference/` directory contains 9 detailed spec documents covering data sources, entity types, map layers, database schema, temporal model, and graph fusion. Start with `reference/06-proposed-architecture.md` for the full design, or `CLAUDE.md` for the summary.

## License

TBD
