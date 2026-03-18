## Proposal: Fix trackpoints pagination (keyset)

---

### How the endpoint works now

`GET /api/0.6/trackpoints?bbox=...&page=N`

The endpoint takes a **bounding box** and a page number. From the code - https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/api/tracepoints_controller.rb#L34-L40 

```ruby
      # get all the points
      ordered_points = Tracepoint.bbox(bbox).joins(:trace).where(:gpx_files => { :visibility => %w[trackable identifiable] }).order(:gpx_id => :desc, :trackid => :asc, :timestamp => :asc)
      unordered_points = Tracepoint.bbox(bbox).joins(:trace).where(:gpx_files => { :visibility => %w[public private] }).order("gps_points.latitude", "gps_points.longitude", "gps_points.timestamp")
      @points = ordered_points.union_all(unordered_points).offset(offset).limit(Settings.tracepoints_per_page).preload(:trace)
```

The endpoint first filters by bounding box using a spatial index — so it only works with points inside the requested area. Then it applies `OFFSET` on those filtered results (see [line 37](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/api/tracepoints_controller.rb#L37)).

For small areas this is fine, but for dense areas (like a big city) there can be millions of points inside the bbox — and OFFSET still has to scan and skip all previous pages on every request.

Test in JOSM [gps-osm-plan/05-download-gpx-jOSM.md](gps-osm-plan/05-download-gpx-jOSM.md)


---

### Possible fix

Replace `OFFSET` with **keyset pagination** — instead of skipping rows, the client sends the last ID it got, and the query picks up from there using an index.

```sql
-- instead of: .offset(N)
-- we can use:
WHERE (gpx_id, trackid) > (:last_gpx_id, :last_trackid)
ORDER BY gpx_id DESC, trackid ASC
LIMIT 5000;
```

No rows are skipped — PostgreSQL jumps straight to the right place in the index. Performance stays the same at any page.

Similar aproach has been applied `/api/0.6/gpx_files` endpoint -  https://github.com/openstreetmap/openstreetmap-website/pull/4089

---

### What  coudl change.

- API parameter: `?page=N` → `?after=GPX_ID:TRACKID` or something similar
- Rails query: `.offset(N)` → `.where("(gpx_id, trackid) > (?, ?)", ...)`
