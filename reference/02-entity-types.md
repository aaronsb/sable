# Entity Types

Every type of object the platform tracks, with classification details.

## Aviation

| Type | Classification Method | Examples |
|------|----------------------|----------|
| Commercial flights | Airline code + route data | QXE2192 (Portland → Anchorage) |
| Private/GA | General aviation type codes | Cessna, Piper, light aircraft |
| Private jets | Business jet model codes | Gulfstream, Bombardier, Citation |
| Military flights | Squawk codes + callsign patterns | Tankers, fighters, reconnaissance |
| Tracked/alert aircraft | Lookup against alert database | POTUS, military special ops |
| UAVs | Hardcoded type signatures | RQ-4 Global Hawk, MQ-9 Reaper, TB2 Bayraktar |

**Properties:** callsign, registration, ICAO24, model, altitude, heading, speed (knots), squawk, origin/destination airports, airline code, GPS position

## Maritime

| Type | Classification Method | Examples |
|------|----------------------|----------|
| Cargo vessels | AIS ship type codes | Container ships, bulk carriers |
| Tankers | AIS type codes | Oil/chemical/LNG tankers |
| Passenger vessels | AIS type codes | Cruise ships, ferries |
| Military vessels | MMSI prefix patterns | Destroyers, frigates, submarines |
| Fishing vessels | AIS type codes | Trawlers, longliners |
| Superyachts | Enrichment DB lookup | Named vessels with ownership OSINT |
| Carrier Strike Groups | News scraping + fallback DB | CVN-68 through CVN-81 |

**Properties:** MMSI, vessel name, ship type (50+ AIS types), heading, speed, position, destination, ETA

## Space

| Type | Classification Method | Examples |
|------|----------------------|----------|
| Reconnaissance | Mission type database | KH-11, Kondor, TOPAZ |
| Navigation | Catalog | GPS, GLONASS, Galileo |
| Communication | Catalog | Starlink, Iridium |
| Weather | Catalog | NOAA, Meteosat |
| ISS/crewed | Catalog | ISS, Tiangong |

**Properties:** NORAD catalog number, TLE data, computed lat/lng/altitude via SGP4, mission classification

## Geophysical Events

| Type | Source | Key Properties |
|------|--------|---------------|
| Earthquakes | USGS | magnitude, depth, location, time |
| Fire hotspots | NASA FIRMS | radiative power, confidence, satellite, acquisition time |
| GPS jamming zones | Inferred from flight dropouts | severity, affected area |
| Internet outages | IODA | country, severity %, BGP/ping metrics |
| Space weather | NOAA SWPC | Kp index, solar flare class |

## Geopolitical Events

| Type | Source | Key Properties |
|------|--------|---------------|
| News articles | RSS feeds | headline, source, risk score, geolocated position, publish time |
| GDELT events | GDELT GKG v2 | event type, mentions, URLs, geotagged location |
| LiveUAMap events | Web scraping | category, description, location, timestamp |
| Conflict frontlines | DeepState GeoJSON | polygon boundaries, controlling party |

## Infrastructure

| Type | Source | Key Properties |
|------|--------|---------------|
| CCTV cameras | TfL, LTA, Austin TX, NYC DOT | location, media URL (HLS/MJPEG), direction |
| KiwiSDR receivers | KiwiSDR API | location, frequency range, antenna type, active users |
| Data centers | Manual/IP2Location | company, region, coordinates |
| Radio systems | OpenMHz | location, talkgroups, recent recordings |
