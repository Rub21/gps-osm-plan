## Proposal: Migrate GPS Traces to GeoParquet

### Current Problem

The `gps_points` table in PostgreSQL contains **~31.8 billion rows (~1.2 TB)**. This causes:

- **Massive backups** of the main database
- **Slow queries** — a bbox query in JOSM takes ~43 seconds
- **B-tree indexes** for 2D spatial queries
- **OFFSET-based pagination** that degrades on later pages
- **Expensive XML serialization** per request

### Solution: Partitioned GeoParquet on S3

Convert the 31.8B rows to **GeoParquet** files compressed with Zstandard, partitioned by geographic grids, stored on S3. Queries served by **DuckDB** (embedded library in Rails, not a separate server).

#### Partitioning strategy: 1°x1° with adaptive subdivision

The base grid is **1°x1°** (~110 km x 110 km at the equator). For areas with high GPS trace density (European cities, Japan, etc.) where the file exceeds ~200 MB, it is subdivided to **0.5°x0.5°** or **0.25°x0.25°**. Areas with no data (oceans, deserts) simply have no file.

- ~64,800 possible cells, in practice ~10,000-15,000 with data
- Grid size is determined during the initial Spark conversion by analyzing actual density
- If a partition grows too large in the future, only that zone is re-partitioned

```
═══════════════════════════════════════════════════════════════
  PROPOSED ARCHITECTURE
═══════════════════════════════════════════════════════════════

  [Current PostgreSQL]           [S3 — GeoParquet]
  gps_points: 31.8B rows  ──►   gps_traces/
  ~1.2 TB with indexes           ├── lat=-45/lon=169/data.parquet       ← 30 MB (normal zone)
                                 ├── lat=48/lon=2/                      ← Paris (dense zone)
                                 │     ├── lat=48.0/lon=2.0/data.parquet  ← subdivided
                                 │     ├── lat=48.0/lon=2.5/data.parquet
                                 │     ├── lat=48.5/lon=2.0/data.parquet
                                 │     └── lat=48.5/lon=2.5/data.parquet
                                 ├── lat=52/lon=13/                      ← Berlin (dense zone)
                                 │     └── ...                           ← subdivided
                                 └── ...
                                 ~80-150 GB total (zstd compression)

  [API /api/0.6/trackpoints?bbox=...]
        │
        ▼
  DuckDB (embedded in Rails): reads only the relevant partitions
        │  downloads only the needed file from S3
        │  uses row group statistics to skip irrelevant blocks
        ▼
  Filters by exact bbox → responds in <1 second
```

### Why GeoParquet?

| Factor | Current PostgreSQL | GeoParquet on S3 |
|---|---|---|
| **Size** | ~1.2 TB | ~80-150 GB (zstd compression) |
| **Bbox query** | ~43 seconds (B-tree, OFFSET) | <1 second (partitioning + DuckDB) |
| **Storage cost** | PostgreSQL server 24/7 | S3: ~$2/month per 100 GB |
| **Backups** | Massive, slow | Static files on S3 (versioned) |
| **Scalability** | Requires more powerful server | Partial reads, no dedicated server |

### Why not FlatGeobuf?

FlatGeobuf does not scale for 31.8B rows:
- No compression (~800+ GB)
- Requires building R-tree in memory on write
- No native partitioning support
- Designed for web visualization, not massive analytics

GeoParquet was specifically designed for this volume of data.
