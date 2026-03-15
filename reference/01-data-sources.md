# Data Sources Inventory

Every external data source the platform connects to, organized by update cadence.

## Fast Tier (60s polling)

### Commercial Flights
- **Source:** ADS-B via adsb.lol, opensky-network.org
- **Auth:** OpenSky OAuth2 (OPENSKY_CLIENT_ID, OPENSKY_CLIENT_SECRET)
- **Data:** Real-time aircraft positions, callsigns, altitude, heading, speed, origin/destination routes
- **Supplemental feeds:** Regional gap-filling from underreported areas (Yekaterinburg, Novosibirsk, Lagos-Accra corridor)

### Military Flights
- **Source:** ADS-B filtered by military squawk codes + callsign patterns
- **Auth:** None (public ADS-B)
- **Data:** Fighter jets, tankers, cargo, reconnaissance, helicopters classified by type codes
- **Includes:** UAV detection (RQ-4 Global Hawk, MQ-9 Reaper, TB2 Bayraktar via hardcoded type signatures)

### Maritime / AIS
- **Source:** AIS WebSocket stream via aisstream.io
- **Auth:** AIS_API_KEY
- **Data:** Real-time vessel positions (tankers, cargo, passenger, military), heading, speed, MMSI
- **Feature:** Viewport-aware streaming — server filters to visible map bounds + 10% padding

### Carrier Strike Groups
- **Source:** GDELT News API + hardcoded fallback database
- **Auth:** None
- **Data:** US Navy carrier positions (CVN-68 through CVN-81) via news scraping

### Satellites
- **Source:** CelesTrak TLE (Two-Line Elements) + SGP4 orbital propagation
- **Auth:** User-Agent header (fair use: 1 req/24h)
- **Data:** 200+ satellite positions classified by mission type (recon, commercial, navigation, ISS)

## Slow Tier (5 minute polling)

### News
- **Source:** RSS feeds (configurable, default 20 feeds) with custom weights
- **Auth:** None
- **Data:** Geolocated news articles with risk scoring and cluster aggregation

### Earthquakes
- **Source:** USGS Earthquake Hazards Program (GeoJSON)
- **Auth:** None
- **Data:** Seismic events magnitude 2.5+, depth, location

### NASA FIRMS Fires
- **Source:** NASA FIRMS CSV (NOAA-20 VIIRS 24h)
- **Auth:** None
- **Data:** Active fire hotspots, fire radiative power, confidence, acquisition time

### Financial
- **Source:** Yahoo Finance / Alpha Vantage
- **Auth:** None
- **Data:** Defense stocks (Lockheed Martin, Raytheon, etc.), Brent/WTI crude oil

### Weather
- **Source:** RainViewer API + NASA GIBS
- **Auth:** None
- **Data:** Radar/satellite weather imagery with historical playback

### Space Weather
- **Source:** NOAA Space Weather Prediction Center (SWPC)
- **Auth:** None
- **Data:** Kp Index, solar flares, geomagnetic storms

### Internet Outages
- **Source:** IODA (Georgia Tech) + Nominatim geocoding
- **Auth:** None
- **Data:** Regional BGP/ping anomalies, outage severity per country

### CCTV Cameras
- **Source:** TfL JamCams (UK), LTA Singapore, Austin TX DOT, NYC DOT
- **Auth:** LTA_ACCOUNT_KEY (optional, for Singapore)
- **Data:** Live traffic camera feeds (HLS/MJPEG), location, direction

### KiwiSDR Receivers
- **Source:** KiwiSDR API
- **Auth:** None
- **Data:** Software-defined radio receiver locations, frequency coverage, active user counts

### Frontlines (Ukraine)
- **Source:** LiveUAMap + DeepState GeoJSON
- **Auth:** None
- **Data:** Conflict frontline polygons, territorial control boundaries

### GDELT Events
- **Source:** GDELT Global Knowledge Graph v2
- **Auth:** None
- **Data:** Global events with geotagging, mentions, URLs, headlines (15-min refresh)

### Data Centers
- **Source:** IP2Location / manual CSV
- **Auth:** None
- **Data:** Critical infrastructure locations (AWS, Google, Azure regions)

## On-Demand (user-triggered)

### Flight Routes
- **Source:** adsb.lol routeset API
- **Data:** Origin/destination airports for any callsign

### Region Dossier
- **Source:** Overpass API (OSM) + reverse geocoding
- **Data:** Administrative regions, populated places, demographics at clicked location

### Sentinel-2 Satellite Imagery
- **Source:** Copernicus/SciHub
- **Data:** High-res satellite imagery by lat/lng, cloud cover, acquisition date

### Radio Feeds
- **Source:** Broadcastify (web scraping) + OpenMHz API
- **Data:** Top 50 emergency dispatch feeds, trunked radio systems, recent call recordings

### Plane/Yacht Alert Enrichment
- **Source:** Local JSON databases (plane_alert_db.json, yacht_alert_db.json)
- **Data:** Special interest aircraft (POTUS, military, special ops), superyacht tracking
