# How Overture Maps uses GeoParquet

## What is Overture?

Overture Maps is an open map data project from Amazon, Meta, Microsoft and TomTom. They publish a global dataset with buildings, roads, places, etc.

They don't use a database. All their data is GeoParquet files on S3.

## How it works

They store everything as Parquet files in S3, organized by theme:

```
s3://overturemaps-us-west-2/release/2026-02-18.0/
├── theme=buildings/type=building/       (~120 GB, 2.3B buildings)
├── theme=transportation/type=segment/   (~240M road segments)
├── theme=places/type=place/
└── ...
```

Every month they publish a new release with the full dataset.

To get data for a specific area you don't need to download all of it. You can query directly from S3 with DuckDB:

```sql
SELECT *
FROM read_parquet('s3://overturemaps-us-west-2/release/.../type=segment/*')
WHERE bbox.xmin > -77.1 AND bbox.xmax < -76.9
  AND bbox.ymin > -12.1 AND bbox.ymax < -11.9;
```

This works because the files are spatially sorted, so the engine skips the parts that are outside your area.

## Why it works for them

The key thing about Overture is that their data is **read-only for users**. Nobody uploads data directly to the Parquet files. They have a build pipeline that generates the full dataset and publishes it.

So they don't need to handle:
- Real-time uploads
- Deleting individual records
- Per-user privacy settings
- Concurrent writes

That is exactly why they can use files instead of a database. The data is static between releases.
