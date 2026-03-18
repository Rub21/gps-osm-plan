# GPS Tile Server

> Related repositories:
> - [`openstreetmap/gpx-updater`](https://github.com/openstreetmap/gpx-updater) — update daemon and tile CGI (Perl)
> - [`e-n-f/datamaps`](https://github.com/e-n-f/datamaps) — spatial index and tile rendering (C)
> - [`e-n-f/gpx-import`](https://github.com/e-n-f/gpx-import) — GPX parser (C)
> - [`openstreetmap/chef`](https://github.com/openstreetmap/chef) — server configuration (cookbook `gps-tile`)

---

`gps-tile.openstreetmap.org` is a server that generates PNG tiles from all public GPS traces uploaded to OSM. It powers the GPS overlay layer on the OSM website and the iD editor.

```
https://gps.tile.openstreetmap.org/lines/{z}/{x}/{y}.png
```

**Server:** `muirdris` — HPE ProLiant DL360 Gen10, Equinix Dublin.

**Privacy:** Only traces with `public` or `identifiable` visibility are included. `trackable` and `private` are excluded.

---

## How it works

There are two separate processes: one that keeps the data up to date, and one that renders tiles on demand.

### Update process

The `gps-update` systemd service runs in a loop every 60 seconds:

1. Fetches the RSS feed: `https://www.openstreetmap.org/traces/rss`
2. Downloads new GPX files: `https://www.openstreetmap.org/trace/{id}/data`
3. Parses each GPX file to extract lat/lon pairs (`gpx-parse` — C tool from [`e-n-f/gpx-import`](https://github.com/e-n-f/gpx-import))
4. Encodes the points into the spatial index (`encode` — C tool from [`e-n-f/datamaps`](https://github.com/e-n-f/datamaps))
5. Merges the new data into the existing index (`merge`) — this is **append-only**, nothing is ever removed
6. Invalidates affected tiles in Memcached so they get re-rendered

### Tile rendering

When a tile is requested:

1. Check Memcached — if cached, serve it directly
2. If not cached, call `render` (C tool from datamaps) to generate a 256x256 PNG from the spatial index
3. Compress with pngquant (64 colors)
4. Store in Memcached and return to client

The color of each line represents the **direction of movement**.

---

## Important: the tile server does not use PostgreSQL

I want to highlight this because it surprised me. The tile server does **not** read from the `gps_points` table. It downloads the GPX files from the OSM website via HTTP and maintains its own independent copy of the data on disk.

This means the same GPS data currently exists in **three separate places**:

| Location | Format | Used for |
|---|---|---|
| AWS S3 | Original GPX files | User downloads |
| PostgreSQL (`gps_points`) | Individual rows | API `/api/0.6/trackpoints` |
| Tile server disk | GPX files + binary spatial index | PNG tile rendering |

---

## Tech stack

The spatial index is powered by [`datamaps`](https://github.com/e-n-f/datamaps) by Erica Fischer. It uses 4 binaries:

| Binary | What it does |
|---|---|
| `encode` | Converts lat/lon pairs into a quadtree spatial index |
| `merge` | Adds new data into the existing index (append-only) |
| `render` | Generates a PNG tile for a given z/x/y |
| `enumerate` | Lists all tiles affected by new data (for cache invalidation) |

Server paths:

| Path | Content |
|---|---|
| `tracks/current/` | Downloaded GPX files (gzipped) |
| `shapes/current/` | Datamaps spatial index (binary files) |

---

## Problems I identified

**Triple data duplication** — the same data lives in S3, PostgreSQL, and the tile server disk. Each has its own ingestion pipeline and no synchronization.

**Cannot delete traces** — the `merge` operation is append-only. If a user deletes a trace or changes its visibility, there is no way to remove it from the tile index. A full rebuild is required, and Grant mentioned it took more than a month the one time it was done.

**RSS polling is fragile** — the daemon discovers new traces by polling an RSS feed. If the daemon is down or if many traces are uploaded at once, traces can be missed.

**Downloads via HTTP instead of S3** — the daemon makes HTTP requests to the OSM website to download GPX files, going through the full Rails stack. It could read directly from S3 instead.

**Old C code with no active maintainer** — `datamaps` and `gpx-parse` are in personal repos by Erica Fischer, not under the OpenStreetMap organization. Grant mentioned she might be willing to help if contacted, but the code has not been updated in ~13 years.

**No filtering** — all public traces are merged into a single dataset. There is no way to filter by date, user, or any other criteria.

---

## Potential improvements

| Aspect | Now | Possible improvement |
|---|---|---|
| Data copies | 3 independent copies | S3 as single source of truth |
| Update discovery | RSS polling every 60s | Webhook or queue triggered on upload |
| GPX download | HTTP via Rails | Direct S3 read |
| Deletions | Not supported | Event-driven index updates |
| Tile format | PNG (raster) | Vector tiles (PMTiles) |
| Filtering | None | By date, user, visibility |
