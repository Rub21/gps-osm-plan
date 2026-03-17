# GPS Tile Server: gps-tile.openstreetmap.org

> Related repositories:
> - [`openstreetmap/gpx-updater`](https://github.com/openstreetmap/gpx-updater) — Update daemon and tile CGI script (Perl)
> - [`e-n-f/datamaps`](https://github.com/e-n-f/datamaps) — Quadtree spatial index and tile rendering (C)
> - [`e-n-f/gpx-import`](https://github.com/e-n-f/gpx-import) — GPX parser that extracts lat/lon pairs (C)
> - [`openstreetmap/chef`](https://github.com/openstreetmap/chef) — Infrastructure configuration (cookbook `gps-tile`)

---

## 1. What is it

`gps-tile.openstreetmap.org` is a dedicated server that generates **rasterized PNG images** from the collection of public GPS traces uploaded to OpenStreetMap. It visualizes the density and direction of movement of trackpoints as colored lines on a transparent background.

**Tile URL:**
```
https://gps.tile.openstreetmap.org/lines/{z}/{x}/{y}.png
```

**Load balancing subdomains:**
- `gps-a.tile`, `gps-b.tile`, `gps-c.tile`
- `a.gps-tile`, `b.gps-tile`, `c.gps-tile`

**Physical server:** `muirdris` — HPE ProLiant DL360 Gen10, located at Equinix Dublin (also runs the OSM Wiki).

**Privacy restriction:** Only includes traces with **public** or **identifiable** visibility. Traces with `trackable` and `private` visibility are excluded to respect user privacy.

---

## 2. How it is used on openstreetmap.org

The tile server feeds the "GPS Traces" overlay layer on the OSM website and the iD editor.

### Frontend integration

| File | Function |
|---|---|
| [`vendor/assets/leaflet/leaflet.osm.js`](https://github.com/openstreetmap/openstreetmap-website/blob/master/vendor/assets/leaflet/leaflet.osm.js) | Defines `L.OSM.GPS` as a Leaflet TileLayer pointing to `gps.tile.openstreetmap.org/lines/{z}/{x}/{y}.png` with `maxNativeZoom: 20`, `maxZoom: 21` |
| [`app/assets/javascripts/leaflet.map.js`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/assets/javascripts/leaflet.map.js) | Instantiates `map.gpsLayer` with layer code `"G"` |
| [`app/controllers/layers_panes_controller.rb`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/layers_panes_controller.rb) | Registers the layer as a checkbox in the layers panel: `{ :layer_id => "gpsLayer", :name => "gps" }` |

**URL activation:** The GPS layer can be activated by adding the code `G` to the layers parameter in the URL.

---

## 3. Tile Generation Pipeline — Step by Step

The tile generation process involves two independent pipelines: an **update pipeline** (ingests new GPS data) and a **rendering pipeline** (serves tile images on demand).

### 3.1 Update Pipeline — `gps-update` daemon

**Repository:** [`openstreetmap/gpx-updater`](https://github.com/openstreetmap/gpx-updater) — file: [`update`](https://github.com/openstreetmap/gpx-updater/blob/master/update)

The daemon runs as a systemd service (`gps-update`) in an infinite loop:

```
Step 1: DISCOVER NEW TRACES
─────────────────────────────────────────────────────
  The daemon fetches the RSS feed:
    GET https://www.openstreetmap.org/traces/rss

  It parses the XML to extract trace IDs.
  Only traces not yet processed are selected.

Step 2: DOWNLOAD GPX FILES
─────────────────────────────────────────────────────
  For each new trace, downloads the raw GPX file:
    GET https://www.openstreetmap.org/trace/{id}/data

  - Downloads run for up to 10 minutes per cycle
  - Only public + identifiable traces are included
  - Files are saved as gzipped GPX:
    /srv/gps-tile.openstreetmap.org/tracks/current/{id}.gpx.gz

Step 3: PARSE GPX → LAT/LON PAIRS
─────────────────────────────────────────────────────
  Tool: gpx-parse (from e-n-f/gpx-import repo)
  Repository: https://github.com/e-n-f/gpx-import
  Language: C (uses libexpat for XML parsing)

  gpx-parse reads each GPX file and outputs raw lat/lon pairs
  to stdout, one per line. It strips all metadata (timestamps,
  altitude, etc.) — only coordinates are kept.

  For directional data, it also calculates the bearing (vector)
  between consecutive points.

Step 4: ENCODE INTO QUADTREE INDEX
─────────────────────────────────────────────────────
  Tool: encode (from e-n-f/datamaps repo)
  Repository: https://github.com/e-n-f/datamaps
  Language: C

  The lat/lon pairs are piped into `encode`, which builds a
  quadtree spatial index stored as binary files on disk.

  The quadtree recursively subdivides the world into quadrants.
  Each leaf node stores the count of points in that area.
  This allows efficient spatial queries at any zoom level.

  Output: binary quadtree files in a temporary directory

Step 5: MERGE WITH EXISTING DATASET
─────────────────────────────────────────────────────
  Tool: merge (from e-n-f/datamaps repo)

  The newly encoded data is merged into the existing spatial
  index at:
    /srv/gps-tile.openstreetmap.org/shapes/current/

  merge combines two datamaps datasets by summing point counts
  at each quadtree node. This is an append-only operation —
  the existing dataset only grows.

Step 6: INVALIDATE AFFECTED TILES
─────────────────────────────────────────────────────
  Tool: enumerate (from e-n-f/datamaps repo)

  enumerate lists all tile coordinates (z/x/y) that are
  affected by the newly merged data. These tile keys are
  then deleted from Memcached, forcing re-render on next
  request.

Step 7: SLEEP AND REPEAT
─────────────────────────────────────────────────────
  The daemon sleeps for 60 seconds, then loops back to Step 1.
  New traces typically appear in tiles within minutes of upload.
```

### 3.2 Rendering Pipeline — Tile CGI

**Repository:** [`openstreetmap/gpx-updater`](https://github.com/openstreetmap/gpx-updater) — file: [`tile`](https://github.com/openstreetmap/gpx-updater/blob/master/tile)

When a client requests a tile (e.g., `GET /lines/12/2048/1024.png`):

```
Step 1: CHECK MEMCACHED
─────────────────────────────────────────────────────
  The CGI script (Perl) checks if the tile exists in Memcached
  using the key: "lines/{z}/{x}/{y}"

  If found → serve cached PNG directly (fastest path)

Step 2: RENDER FROM SPATIAL INDEX
─────────────────────────────────────────────────────
  Tool: render (from e-n-f/datamaps repo)
  Repository: https://github.com/e-n-f/datamaps
  Language: C

  If not cached, the CGI calls `render` with:
    - The quadtree dataset path (shapes/current/)
    - The tile coordinates (z, x, y)
    - Flag -g for directional coloring (lines-directional.dm)

  render traverses the quadtree to find all points within the
  tile's bounding box. It draws colored lines representing GPS
  traces, where color encodes the direction of movement.

  Output: raw PNG image (256x256 pixels, transparent background)

Step 3: COMPRESS WITH PNGQUANT
─────────────────────────────────────────────────────
  The raw PNG is piped through pngquant, which reduces the
  color palette to 64 colors. This significantly reduces file
  size while maintaining visual quality for line data.

Step 4: CACHE AND SERVE
─────────────────────────────────────────────────────
  The compressed PNG is:
    1. Stored in Memcached for future requests
    2. Returned to the client with Content-Type: image/png
```

---

## 4. Technology Stack — Detailed

### 4.1 Components

| Component | Technology | Repository | Key files |
|---|---|---|---|
| Update daemon | Perl | [`openstreetmap/gpx-updater`](https://github.com/openstreetmap/gpx-updater) | [`update`](https://github.com/openstreetmap/gpx-updater/blob/master/update) |
| Tile CGI | Perl | [`openstreetmap/gpx-updater`](https://github.com/openstreetmap/gpx-updater) | [`tile`](https://github.com/openstreetmap/gpx-updater/blob/master/tile) |
| GPX parser | C (libexpat) | [`e-n-f/gpx-import`](https://github.com/e-n-f/gpx-import) | [`src/gpx-parse.c`](https://github.com/e-n-f/gpx-import/blob/master/src/gpx-parse.c) |
| Spatial index (encode) | C | [`e-n-f/datamaps`](https://github.com/e-n-f/datamaps) | [`encode.c`](https://github.com/e-n-f/datamaps/blob/master/encode.c) |
| Tile renderer | C | [`e-n-f/datamaps`](https://github.com/e-n-f/datamaps) | [`render.c`](https://github.com/e-n-f/datamaps/blob/master/render.c) |
| Dataset merge | C | [`e-n-f/datamaps`](https://github.com/e-n-f/datamaps) | [`merge.c`](https://github.com/e-n-f/datamaps/blob/master/merge.c) |
| Tile enumerator | C | [`e-n-f/datamaps`](https://github.com/e-n-f/datamaps) | [`enumerate.c`](https://github.com/e-n-f/datamaps/blob/master/enumerate.c) |
| HTTP server | Apache + mod_cgid | — | — |
| Cache | Memcached | — | — |
| PNG compression | pngquant (64 colors) | — | — |

### 4.2 `datamaps` — The spatial engine

[`e-n-f/datamaps`](https://github.com/e-n-f/datamaps) by Eric Fischer is the core engine powering the tile server. It provides four key binaries:

| Binary | Purpose | Input | Output |
|---|---|---|---|
| `encode` | Builds a quadtree spatial index from lat/lon pairs | stdin (lat/lon text) | Binary quadtree files on disk |
| `merge` | Combines two datasets by summing point counts | Two dataset directories | Merged dataset directory |
| `render` | Generates a PNG tile from the spatial index | Dataset + z/x/y coords | PNG image (stdout) |
| `enumerate` | Lists all tile coordinates that contain data | Dataset directory | z/x/y coordinates (stdout) |

**Quadtree structure:** The index recursively divides the world into quadrants. At each level, a node either contains a point count or references to four child nodes. This allows O(log n) spatial lookups regardless of dataset size.

**Directional rendering:** When the `-g` flag is passed to `render`, it reads directional data (bearing between consecutive points) and maps it to colors using a color wheel. This is what produces the colored lines visible on the GPS layer.

### 4.3 `gpx-parse` — The GPX parser

[`e-n-f/gpx-import`](https://github.com/e-n-f/gpx-import) contains `gpx-parse`, a lightweight C program that:

1. Reads GPX XML using libexpat (SAX parser, low memory)
2. Extracts `<trkpt lat="..." lon="...">` elements
3. Outputs lat/lon pairs (and optionally bearing) to stdout
4. Handles compressed files (gzip, bzip2)

This is the **same repository** as the old gpx-import daemon, but only the `gpx-parse` binary is used by the tile server. The daemon portion is unused.

### 4.4 Server paths

| Path | Content |
|---|---|
| `/srv/gps-tile.openstreetmap.org/tracks/current/` | Downloaded GPX files (gzipped) |
| `/srv/gps-tile.openstreetmap.org/shapes/current/` | Datamaps spatial index (quadtree binary files) |
| `/srv/gps-tile.openstreetmap.org/` | Service home directory |

### 4.5 Chef configuration

| File | Function |
|---|---|
| [`cookbooks/gps-tile/recipes/default.rb`](https://github.com/openstreetmap/chef/blob/master/cookbooks/gps-tile/recipes/default.rb) | Main recipe: installs dependencies, clones repos, compiles C code, configures services |
| [`roles/gps-tile.rb`](https://github.com/openstreetmap/chef/blob/master/roles/gps-tile.rb) | Server role definition |
| [`cookbooks/gps-tile/templates/default/apache.erb`](https://github.com/openstreetmap/chef/blob/master/cookbooks/gps-tile/templates/default/apache.erb) | Apache configuration with CGI |

- Service user: `gpstile` (UID 519)
- Systemd service: `gps-update`
- SSL configured for all domain variants

---

## 5. Data Source: Independent from PostgreSQL

**Critical fact:** The tile server **does NOT read from the `tracepoints` table in PostgreSQL.** It maintains its own completely independent copy of the data.

### Data flow diagram

```
                    ┌──────────────┐
                    │  User        │
                    │  uploads GPX │
                    └──────┬───────┘
                           │
                           v
                    ┌──────────────┐
                    │  Rails App   │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              v            v            v
        ┌──────────┐ ┌──────────┐ ┌──────────────┐
        │   S3     │ │PostgreSQL│ │  RSS feed    │
        │  (original│ │tracepoints│ │  /traces/rss │
        │   GPX)   │ │(row by   │ └──────┬───────┘
        └──────────┘ │ row)     │        │
                     └──────────┘        v
                                  ┌──────────────┐
                                  │ gps-update   │
                                  │ daemon       │
                                  │ (downloads   │
                                  │ GPX via HTTP)│
                                  └──────┬───────┘
                                         v
                                  ┌──────────────┐
                                  │ Local copy   │
                                  │ GPX + index  │
                                  │ datamaps     │
                                  └──────────────┘
```

This means there are currently **three independent copies** of the GPS data:

| Location | Format | Purpose | Size estimate |
|---|---|---|---|
| **AWS S3** | Original GPX files | ActiveStorage, user downloads | All uploaded files |
| **PostgreSQL** (`tracepoints`) | Individual rows (lat, lon, altitude, timestamp) | API `/api/0.6/trackpoints` | ~31.8 billion rows ([source](https://planet.openstreetmap.org/statistics/data_stats.html)) |
| **Tile server** (local disk) | Gzipped GPX + quadtree binary index | PNG tile rendering | Full dataset duplicate |

---

## 6. Tile format

- **Format:** PNG, 256x256 pixels
- **Background:** Transparent (overlay on top of other layers)
- **Colors:** Represent the **direction of movement** (flag `-g` in render, using `lines-directional.dm`)
- **Compression:** pngquant reduces to 64 colors for smaller size
- **Zoom:** Native up to zoom 20, overzoom up to 21

---

## 7. Identified Problems

### 7.1 Triple data duplication
The same GPS data is stored in three completely independent systems (S3, PostgreSQL, tile server disk). Each has its own ingestion pipeline, its own format, and no synchronization between them. This wastes storage, increases operational complexity, and creates potential inconsistencies.

### 7.2 RSS-based discovery is fragile
The tile server discovers new traces by polling an RSS feed every 60 seconds. RSS feeds are typically limited to the most recent entries. If the daemon is down for an extended period or if many traces are uploaded in a burst, traces could be missed entirely.

### 7.3 Download via HTTP instead of S3
The daemon downloads GPX files by making HTTP requests to the OSM website (`/trace/{id}/data`), which goes through the full Rails stack. It could read directly from S3, bypassing the application server entirely and reducing load on the web servers.

### 7.4 Legacy stack with single maintainer risk
The core spatial engine (`datamaps`) and GPX parser (`gpx-parse`) are C programs in personal repositories (`e-n-f/`) by Eric Fischer. These are not under the OpenStreetMap organization and have limited community maintenance. The update daemon and tile CGI are Perl scripts served via Apache mod_cgid.

### 7.5 Raster tiles only (PNG)
Tiles are pre-rendered as PNG images. This means:
- No client-side styling or interaction
- Fixed visual representation regardless of context
- Larger file sizes compared to vector tiles
- Cannot be queried (e.g., "show me only traces from the last year")

### 7.6 No filtering or temporal queries
The tile server renders **all** public traces as a single merged dataset. There is no way to:
- Filter by date range
- Filter by user
- Filter by trace quality or density
- Show only recent traces

### 7.7 Append-only index cannot handle deletions
The datamaps merge operation only adds data. If a user deletes a trace or changes its visibility from public to private, the tile server has no mechanism to remove those points from the spatial index. A full rebuild would be required.

### 7.8 On-demand rendering under CGI
Each uncached tile request spawns a new CGI process (Perl) which then forks `render` (C). This is the slowest possible web serving model. Modern alternatives (pre-generated tiles, long-lived processes, or vector tiles served from S3) would be significantly more efficient.

---

## 8. Potential Improvements

| Aspect | Current state | Potential improvement |
|---|---|---|
| **Tile format** | Rasterized PNG (256x256) | Vector tiles (MVT/PMTiles) |
| **Storage** | Local disk + Memcached | S3 + CDN (CloudFront) |
| **Spatial index** | datamaps (proprietary quadtree) | PMTiles, MBTiles, or PostGIS |
| **Data source** | Downloads GPX via HTTP/RSS | Direct S3 read or event-driven (SQS/webhook) |
| **Update mechanism** | Polling RSS every 60s | Webhook or message queue triggered on upload |
| **Rendering** | On-demand CGI per request | Pre-generated tiles or long-lived tile server |
| **Filtering** | None (all traces merged) | Temporal and user-based filtering |
| **Deletions** | Not supported (append-only) | Event-driven index updates |
| **Data copies** | 3 independent copies | Unified storage (S3 as source of truth) |
