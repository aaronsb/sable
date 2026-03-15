# Architecture Anti-Patterns in Original

What to avoid in the rebuild. Every item here is a concrete problem observed in the original codebase.

## No Database
- All state lives in an in-memory Python dict (`latest_data` in `_store.py`)
- State is completely lost on restart
- No historical data, no replay, no time-series queries
- JSON file caches on disk for some reference data, but no indexing
- **Fix:** Postgres with proper schemas, historical event tables, spatial indexing (PostGIS)

## Monolithic Frontend
- `MaplibreViewer.tsx` = 2,302 lines (map + all visual logic + GeoJSON builders + popups + interpolation)
- `NewsFeed.tsx` = 1,089 lines
- No component decomposition, rendering logic intertwined with data fetching
- **Fix:** Decompose into layer components, separate data hooks from rendering

## Monolithic Backend
- `main.py` = 522 lines with 30+ endpoints, lifespan management, CORS, auth all in one file
- All data sources fetched by one scheduler in one process
- No microservices, no message queue, no worker separation
- **Fix:** FastAPI routers, pluggable collector modules, separate scheduler from API

## No Pagination
- All list endpoints return full arrays
- Flight data can be thousands of objects per response
- No cursor, no limit, no offset
- **Fix:** Cursor-based pagination on every list endpoint from day one

## Polling Architecture
- Frontend polls backend every 3-5 seconds for fast data
- Backend polls upstream APIs every 60 seconds
- Double-polling with no event-driven path
- ETag caching helps but doesn't solve the fundamental inefficiency
- **Fix:** WebSocket or SSE for real-time push, reduce poll-based updates to slow-tier only

## .env File Secrets
- API keys managed via `.env` files and environment variables
- Settings modal lets users edit keys via API (stored back to `.env` on disk)
- No encryption at rest, no vault integration
- **Fix:** Docker secrets, Vault, or at minimum encrypted config store

## No API Versioning
- All endpoints under `/api/` with no version prefix
- Breaking changes would affect all clients
- **Fix:** `/v1/` prefix from the start

## Hardcoded Classification Logic
- Military flight detection via hardcoded callsign patterns and squawk codes
- UAV detection via hardcoded model strings
- Carrier positions via hardcoded fallback coordinates
- **Fix:** Classification rules as configurable data, not code

## Self-Update Mechanism
- Backend has a built-in updater that downloads from GitHub releases and applies patches
- Runs with same permissions as the application
- **Fix:** Remove entirely. Use standard deployment (container rebuild, git pull, CI/CD)

## Frontend Performance
- Re-renders entire map on every poll cycle
- No virtualization on list panels (news, radio feeds)
- GeoJSON rebuilt from scratch on each update instead of diffed
- Sub-frame interpolation computed for all entities, not just visible ones
- **Fix:** Incremental GeoJSON updates, viewport-aware rendering, list virtualization
