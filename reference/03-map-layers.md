# Map Layers and Visualization

48+ controllable layers rendered on a MapLibre GL map.

## Base Map
- **Tiles:** CartoDB (4 CDN mirrors), dark and light variants
- **Fonts:** Arial Unicode Bold, OpenSans Bold (self-hosted PBF glyphs)
- **Style:** Custom JSON map style

## Entity Layers

### Aviation
| Layer | Icon Style | Color Coding |
|-------|-----------|--------------|
| Commercial flights | Aircraft silhouette + heading rotation | 11 color variants by airline/operator |
| Private/GA | Smaller aircraft silhouette | Cyan, orange, purple, yellow, pink |
| Military | Fighter/helicopter/tanker silhouettes | Red variants, special POTUS icon |
| UAVs | Drone silhouettes | By type (Global Hawk, Reaper, Bayraktar) |
| Tracked/alert | Alert-boxed aircraft | Red/pink with wiki link popup |
| Flight trails | Polyline behind aircraft | Fades with age |
| Flight routes | Origin → destination line | Drawn on click |

### Maritime
| Layer | Icon Style | Color Coding |
|-------|-----------|--------------|
| Cargo/tanker | Ship silhouette | Gray/yellow |
| Military vessels | Warship silhouette | Red |
| Passenger | Ship silhouette | Blue |
| Superyachts | Yacht silhouette | Pink |
| Carrier groups | Special carrier icon | Red with group name |

### Satellites
| Layer | Icon Style | Color Coding |
|-------|-----------|--------------|
| All satellites | Dynamic SVG by mission | Red=military, blue=commercial, green=navigation |

### Geophysical
| Layer | Icon Style | Notes |
|-------|-----------|-------|
| Earthquakes | Triangle markers | Yellow/red by magnitude, size scales |
| FIRMS fires | Heat gradient circles | 4 severity levels (yellow → dark red), clustering |
| GPS jamming | Warning triangles | Yellow/red by severity |
| Internet outages | Region heat overlay | Color by outage percentage |

### Geopolitical
| Layer | Icon Style | Notes |
|-------|-----------|-------|
| News articles | Alert pin markers | Color by risk score |
| GDELT events | Clustered points | Drill-down on cluster click |
| LiveUAMap events | Category icons | Description popups |
| Frontlines | Filled polygons | Red/blue by territorial control |

### Infrastructure
| Layer | Icon Style | Notes |
|-------|-----------|-------|
| CCTV cameras | Video camera icon | Click for live HLS/MJPEG |
| KiwiSDR | Radio tower icon | Frequency/user metadata tooltip |
| Data centers | Server cluster icon | Company + region metadata |

## Overlay Layers
| Layer | Type | Source |
|-------|------|--------|
| Weather radar/satellite | TMS raster overlay | NASA GIBS with opacity + date slider |
| Sentinel-2 imagery | WMS raster overlay | Copernicus CloudFront CDN |
| Day/night terminator | Computed polygon | Solar calculation, recomputed every 60s |
| Submarine cables | GeoJSON polylines | Static dataset (cables.geojson) |

## Map Interaction
- **Click entity:** Detail panel with properties, photos, wiki links
- **Right-click:** Context menu (region dossier, nearest radio, measure from here)
- **Viewport scroll:** Server-side AIS stream filtering updates
- **Eavesdrop mode:** Click location → find nearest radio system + recent calls
- **Measure tool:** Click-to-click distance computation
- **Flight route preview:** Click aircraft → fetch and draw origin/destination
- **Trail interpolation:** Smooth motion between 60s poll updates with sub-frame interpolation
