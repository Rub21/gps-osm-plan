## Proposal: Migrate GPS Traces to GeoParquet on S3 ~~(Not recommended)~~

### The problem

The `gps_points` table in PostgreSQL has ~31.8 billion rows (~1.2 TB). This causes:

- Massive database backups
- Slow bbox queries — a JOSM query takes ~43 seconds
- OFFSET-based pagination that gets worse on later pages

---

### The idea

Convert the 31.8B rows to **GeoParquet** files stored on S3, partitioned by geographic grid. Queries would be served by **DuckDB** embedded in Rails

---

### How Overture Maps does it (reference)

Overture Maps (Amazon, Meta, Microsoft, TomTom) stores all their global data — 2.3B buildings, 240M road segments — as GeoParquet on S3 with no database. See: [overture-geoparquet-reference.md](../docs/overture-geoparquet-reference.md)

Their key trick is **spatial sorting + row group statistics**:

- Data inside each file is sorted by geographic position (using a Hilbert curve or bbox columns)
- Parquet stores min/max statistics per row group
- DuckDB uses those statistics to **skip row groups that are outside the requested bbox**, without downloading the whole file

```sql
-- DuckDB only downloads the row groups that overlap the bbox
SELECT * FROM read_parquet('s3://bucket/gps_traces/lat=48/lon=2/*.parquet')
WHERE lat BETWEEN 48.1 AND 48.3
  AND lon BETWEEN 2.1 AND 2.4;
```

This is how they avoid downloading large files for small queries — even a 30MB partition file might only require reading 1-2 MB if the data is well sorted.

---

### Partitioning strategy

The API has a max bbox of `0.25` square degrees ([settings.yml](https://github.com/openstreetmap/openstreetmap-website/blob/master/config/settings.yml#L31)), which is roughly `0.5°×0.5°`. This is useful for choosing partition size.

I think the grid should be adaptive based on density:

| Area | Grid size | Reason |
|---|---|---|
| Sparse (oceans, deserts) | `1°×1°` | Few points, small file |
| Normal | `0.5°×0.5°` | Matches API max bbox |
| Dense (London, Tokyo, Berlin) | `0.25°×0.25°` | Keep files manageable |

How many files a request touches:
- A small bbox (typical JOSM edit area) → usually **1 file**
- A max-size bbox (`0.5°×0.5°`) → **1 file** if aligned, up to **4 files** in the worst case (bbox falls on a corner between 4 cells)

Within each file, points are sorted by Hilbert curve so DuckDB can skip row groups — even for a 1°×1° file, a small bbox query only reads ~1-2 row groups instead of the whole file.

Total estimated size: ~80-150 GB (zstd compression vs ~1.2 TB in PostgreSQL).


### Conclusion

After reading and finding references, GeoParquet does not seem like a good fit for this project:
- It requires extra work to set up the development environment.
- GeoParquet is difficult to update or delete trace points — you have to download the whole partition file, make the change, and upload it again.
