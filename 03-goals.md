## Goals

From the [OSM software roadmap](https://github.com/openstreetmap/software-roadmap#operational-sustainability), the proposal defines two tracks of work:

### Tasks for the external contractor (STF-funded) — in priority order:
1. Design and implement a process to reconstitute only the necessary parts of GPS uploads
2. Design and implement a new tile rendering pipeline in a more maintainable language
3. Design and implement a way to purge redacted traces from GPS tiles
4. Update [planet-gpx-dump](https://github.com/openstreetmap/planet-gpx-dump) to filter by permissions and run it again ([operations issue #258](https://github.com/openstreetmap/operations/issues/258))

### Tasks:
1. Migrate GPS trackpoints to a separate database referenced from the main one
2. Design a better way to paginate trackpoints by bounding box
3. Reimplement trace animations using a maintained library (not libgd2)

### Design notes
- Better pagination would require an API-breaking change, but there may be ways to keep backward compatibility
- GPS tiles could stay as raster or migrate to vector format — vector optimization is difficult but Mapillary has done it
- Trace animations don't need to be GIFs — openstreetmap-ng uses animated SVGs, and MapLibre can do [JavaScript animations](https://maplibre.org/maplibre-gl-js/docs/examples/animate-a-line/)
- There is a possibility of integrating GPS trace listing directly into the main slippy map
