## Ideas: PostgreSQL + PostGIS for gps_points

### How spatial filtering works  in the wbesite

The `gps_points` table stores lat/lon as integers and has a `tile` column with a B-tree index.

Current table — https://github.com/openstreetmap/openstreetmap-website/blob/master/app/models/tracepoint.rb

```
gps_points
  latitude   integer
  longitude  integer
  tile       bigint      ← B-tree index — used to speed up bbox selection
  gpx_id     bigint
  trackid    integer
  timestamp  datetime
  altitude   float
```

When a bbox request comes in, the query uses QuadTile to pre-filter rows, then checks the exact lat/lon — https://github.com/openstreetmap/openstreetmap-website/blob/master/lib/osm.rb#L522-L527

```ruby
  # Return an SQL fragment to select a given area of the globe
  def self.sql_for_area(bbox, prefix = nil)
    tilesql = QuadTile.sql_for_area(bbox, prefix)
    bbox = bbox.to_scaled

    "#{tilesql} AND #{prefix}latitude BETWEEN #{bbox.min_lat} AND #{bbox.max_lat} " \
      "AND #{prefix}longitude BETWEEN #{bbox.min_lon} AND #{bbox.max_lon}"
  end

```

The `tile` column already helps: points close on the map get similar tile numbers, so the B-tree index can quickly narrow down candidates before checking the exact lat/lon.

---

### Idea A: add a PostGIS geometry column (keep individual points)

Pablo mentioned this in the call — instead of storing the 31B rows as plain integers, we could convert each point to a PostGIS geometry object. Bbox queries would then use a GIST index, which could be a bit faster. He was not certain though — his words were "el performance sería un poco mejor todavía pienso".

The idea is to add a `geom` column with a GIST index:

```sql
geom  geometry(Point, 4326)
```

The bbox query would then use:

```sql
SELECT * FROM gps_points
WHERE ST_Within(geom, ST_MakeEnvelope(:min_lon, :min_lat, :max_lon, :max_lat, 4326))
ORDER BY gpx_id DESC, trackid ASC
LIMIT 5000;
```

**Pros:**
- No change to row count — simpler migration
- PostGIS is already being added to the codebase (PR #6713)
- GIST is designed for spatial queries, better than QuadTile B-tree

**Cons:**
- A PostGIS `Point` object is heavier than two integers. At 31B rows, even a few extra bytes per row means a lot more disk usage.
- To store altitude and timestamp I would need `POINTZM` — even heavier per row.
- Not sure if GIST on 31B rows is fast enough for a bbox query at this scale. Needs testing.
- Still 31B rows, backup size does not improve.

---

### Idea B: store traces as LineString (reduce row count)

Pablo proposed this in an email — instead of storing 31B individual points, store each GPS trace as a single `LINESTRING` geometry. This would reduce the table from 31B rows to roughly the number of traces (~millions).

Pablo tested it:

```sql
CREATE TABLE test_traces (
    name CHARACTER VARYING NOT NULL,
    trace public.geometry(GEOMETRY,4326) NOT NULL
);

INSERT INTO test_traces (name, trace) VALUES
  ('Trace with long steps',  'LINESTRING(0 0, 5 0)'),
  ('Trace with short steps', 'LINESTRING(0 0, 1 0, 2 0, 3 0, 4 0, 5 0)');

SELECT name FROM test_traces
WHERE ST_Crosses(
  test_traces.trace,
  'POLYGON((1.5 -1, 2.5 -1, 2.5 1, 1.5 1, 1.5 -1))'::geometry(GEOMETRY,4326)
);
-- Returns both traces
```

**Pros:**
- Row count drops from 31B to ~millions (one row per trace instead of one per point).
- GIST index on a much smaller table — queries should be significantly faster.
- Can find traces that cross a bbox even without a point inside it (useful for sparse traces with few points per km).
- Fewer rows means faster index operations (reindex, vacuum, etc).

**Cons:**
- `ST_Crosses` only works when the trace crosses the bbox border — also need `ST_Contains` for traces fully inside the bbox.
- JOSM and iD expect individual points, not LineStrings. The query would need to convert the LineString back to points, in SQL or in Rails.
- If I want to store altitude and timestamp per point I would need `LINESTRINGZM` — Pablo noted this was unexpected and adds complexity.
- Disk size does not improve much — the same coordinates are stored, just in a different structure. A `LINESTRINGZM` could even be slightly larger due to object overhead.
- Migration is complex: I would need to group 31B rows by trace and build a LineString per trace — could take several weeks.

---

### Why PostGIS makes sense here

PostGIS is already being introduced in the codebase for the anonymous notes feature (PR #6713, Pablo's current work). So using it for gps_points would not be the first time — the dependency is already there.

---

### Same database or separate database?

Pablo said to keep it in the same database for now. A separate database is possible in the future but adds maintenance complexity. He said to ask Andy and Tom if needed.

Grant is not against keeping GPS points in PostgreSQL if it is the easiest to maintain — "storage is relatively cheap". He also does not know how well Rails handles two databases — "that's more a question for Andy and Tom". I checked and Rails 8.1 does support multiple databases via `connects_to`, but it is not configured yet.

---

### What this fixes

| Issue | Fixed? |
|---|---|
| Slow bbox queries | Possibly (Idea A) / Better (Idea B) |
| Row count (31.8B rows) | No (Idea A) / Yes — fewer rows (Idea B) |
| Pagination timeouts (OFFSET) | No — separate problem |
| Cannot delete traces from tiles | No |
| C code with no maintainers | No |

---

### Open questions

- Is GIST actually faster than QuadTile at 31B rows? I need to test this.
- Idea B: how to handle altitude and timestamp in `LINESTRINGZM`?
- Idea B: migration could take several weeks — need to plan carefully.
