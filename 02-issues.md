# GPS Traces — Known Issues and Goals

From the [proposal document](https://hackmd.io/@1ec5/Hkt6a3P8Ze)

---

## Issues

### Database / Backups

The `gps_points` table has ~31.8 billion rows as of February 2026 (was 22 billion in 2023 — [source](https://x.com/OSM_Tech/status/1684295081199562752)). This table is about ~ 1.2 TB, which makes database backups very large and slow. Stats: https://planet.openstreetmap.org/statistics/data_stats.html

The system also saves the complete original GPX file uploaded by the user, even if it contains private information (like the user's home folder path in the file metadata).

### Rails App

**Pagination:** In 2023, the API for listing traces (`/api/0.6/gpx_files`) was migrated from page numbers to trace IDs ([PR #4089](https://github.com/openstreetmap/openstreetmap-website/pull/4089)). But the trackpoints endpoint (`/api/0.6/trackpoints`) was not migrated and still uses page-number pagination with OFFSET. High page numbers cause timeouts because PostgreSQL has to scan and skip all previous rows. JOSM calls this endpoint in the background and silently ignores the errors — users don't see the failures. API docs: https://wiki.openstreetmap.org/wiki/API_v0.6#GPS_traces

**Massive uploads:** Telemetry systems have been uploading large amounts of traces, causing delays in the import queue for normal users. See [community discussion](https://community.openstreetmap.org/t/gps-trace-upload-delay/104881).

**GIF generation:** The animated GIF generation in the Rails app depends on libgd2, an old library. The Ruby wrapper for it ([gd2-ffij](https://github.com/dark-panda/gd2-ffij)) has an open PR without maintenance: [gd2-ffij PR #28](https://github.com/dark-panda/gd2-ffij/pull/28).

### Tile Renderer

**Cannot delete traces:** If a user deletes or redacts a trace, the tile renderer cannot remove it from the existing tiles. In 2023, the team had to rebuild the entire tileset after deleting some traces — this process took about one month. See [gpx-updater issue #4](https://github.com/openstreetmap/gpx-updater/issues/4).

**No active maintainers:** The tile rendering pipeline ([gpx-updater](https://github.com/openstreetmap/gpx-updater/), [gpx-import](https://github.com/e-n-f/gpx-import/), [datamaps](https://github.com/e-n-f/datamaps/)) is all C code with no one actively maintaining it. Finding someone to fix bugs or add features is difficult.


---

## Goals (STF-funded work)

From the [OSM software roadmap](https://github.com/openstreetmap/software-roadmap#operational-sustainability), the proposal defines two tracks of work:

### Tasks for the external contractor (STF-funded) — in priority order:
1. Design and implement a process to reconstitute only the necessary parts of GPS uploads
2. Design and implement a new tile rendering pipeline in a more maintainable language
3. Design and implement a way to purge redacted traces from GPS tiles
4. Update [planet-gpx-dump](https://github.com/openstreetmap/planet-gpx-dump) to filter by permissions and run it again ([operations issue #258](https://github.com/openstreetmap/operations/issues/258))

### Tasks for the Core Software Engineer (in parallel):
1. Migrate GPS trackpoints to a separate database referenced from the main one
2. Design a better way to paginate trackpoints by bounding box
3. Reimplement trace animations using a maintained library (not libgd2)

### Design notes
- Better pagination would require an API-breaking change, but there may be ways to keep backward compatibility
- GPS tiles could stay as raster or migrate to vector format — vector optimization is difficult but Mapillary has done it
- Trace animations don't need to be GIFs — openstreetmap-ng uses animated SVGs, and MapLibre can do [JavaScript animations](https://maplibre.org/maplibre-gl-js/docs/examples/animate-a-line/)
- There is a possibility of integrating GPS trace listing directly into the main slippy map
