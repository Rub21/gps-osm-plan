# GPS Traces in OpenStreetMap - Current Status

From: https://hackmd.io/@1ec5/Hkt6a3P8Ze

Related repositories:
- [openstreetmap/openstreetmap-website](https://github.com/openstreetmap/openstreetmap-website) — current implementation (upload + import)
- [openstreetmap/gpx-import](https://github.com/openstreetmap/gpx-import) — old C daemon (archived Jul 2022)
- [e-n-f/gpx-import](https://github.com/e-n-f/gpx-import) — personal fork by Erica Fisher, also used for tile rendering
- [e-n-f/datamaps](https://github.com/e-n-f/datamaps) — indexes trackpoints and generates GPS tiles
- [openstreetmap/gpx-updater](https://github.com/openstreetmap/gpx-updater) — monitors RSS feed and downloads public traces for tile rendering
- [openstreetmap/planet-gpx-dump](https://github.com/openstreetmap/planet-gpx-dump) — generates a full dump of public GPS traces (not updated since 2013)
- [openstreetmap/chef](https://github.com/openstreetmap/chef) — infrastructure and configuration

---

## 1. What was gpx-import?

`gpx-import` was the original GPX processing daemon for OpenStreetMap, created in 2008. It was a background process written in C that processed GPX files uploaded by users: it extracted the trace points, inserted them into PostgreSQL, generated preview images, and sent email notifications.

- Written in C (94.5%) — [source code](https://github.com/openstreetmap/gpx-import/tree/master/src)
- License: GNU GPL v2
- Archived by the OSM organization on July 25, 2022
- Related issue: [#1852 - Integrate the high-performance GPX importer](https://github.com/openstreetmap/openstreetmap-website/issues/1852)

The fork by Eric Fisher ([e-n-f/gpx-import](https://github.com/e-n-f/gpx-import)) added direction/vector calculation for each point, but it was never integrated upstream and has no updates since 2013.

---

## 2. Architecture until 2019

The C daemon ran as a systemd service with its own database user (`gpximport`) and direct access to PostgreSQL. It was configured through the Chef cookbook [web::gpx](https://github.com/openstreetmap/chef/blob/245c47e/cookbooks/web/recipes/gpx.rb) (removed in https://github.com/openstreetmap/chef/commit/2a7d30a, Jun 2019.

There was also a dedicated Chef role called [web-gpximport](https://github.com/openstreetmap/chef/blob/a7d96c8/roles/web-gpximport.rb), meaning there was a server just for running this daemon.

The flow was:
1. User uploads a GPX file
2. Rails saves the file to local disk (`/store/rails/gpx/traces`)
3. The C daemon polls every 40 seconds, finds the new file
4. It parses the GPX (using libexpat), inserts points into PostgreSQL (direct connection), generates a preview image (using libgd), and sends an email

---

## 3. The Migration (2019)

In 2019, the C daemon was replaced by Ruby code inside the Rails app. The main author was @gravitystorm, with merges by @tomhughes.

Key PRs:

1. **[PR #2120](https://github.com/openstreetmap/openstreetmap-website/pull/2120)** — Created `TraceImporterJob` and `TraceDestroyerJob` using Rails ActiveJob. This replaced the C daemon. Also introduced preview icon generation using the `gd2-ffij` gem. Resolved issues [#281](https://github.com/openstreetmap/openstreetmap-website/issues/281) and [#1852](https://github.com/openstreetmap/openstreetmap-website/issues/1852).

2. **[PR #2131](https://github.com/openstreetmap/openstreetmap-website/pull/2131)** — Replaced row-by-row insertion with bulk inserts using `activerecord-import` gem (batches of 1,000 points). Much faster even on SSDs.

3. **[PR #2190](https://github.com/openstreetmap/openstreetmap-website/pull/2190)** — Enabled the new system by default.

4. **[PR #2204](https://github.com/openstreetmap/openstreetmap-website/pull/2204)** — Added animated GIF generation showing the trace route point by point.

5. **[PR #2422](https://github.com/openstreetmap/openstreetmap-website/pull/2422)** — Removed the last references to the old C daemon in the code.

On June 10, 2019, Tom Hughes removed the Chef infrastructure for the daemon in https://github.com/openstreetmap/chef/commit/2a7d30a

---

## 4. Current Architecture

GPX processing is now part of https://github.com/openstreetmap/openstreetmap-website.

### Main files

- [traces_controller.rb](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/traces_controller.rb) — handles trace upload and management
- [trace.rb](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/models/trace.rb) — business logic, parsing, and storage
- [trace_importer_job.rb](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/jobs/trace_importer_job.rb) — async import job
- [trace_destroyer_job.rb](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/jobs/trace_destroyer_job.rb) — async deletion job
- Uses the [gpx gem](https://github.com/dougfales/gpx) for parsing GPX files in Ruby

### Data model (table `gpx_files`)

The table is still called `gpx_files` for historical reasons. Main columns: `id`, `user_id`, `visible`, `name`, `size`, `latitude`, `longitude`, `timestamp`, `description`, `inserted` (false = pending import), `visibility` (private, public, trackable, identifiable).

Related tables: `gpx_file_tags` for tags, `gps_points` for the individual trace points.

Files and images are stored in S3 using ActiveStorage:

| Bucket | Content |
|---|---|
| `openstreetmap-gps-traces` | Original GPX files |
| `openstreetmap-gps-images` | Preview images and thumbnails |

Region: `eu-west-1` (Ireland).

### How upload works

1. **Upload**: User uploads a GPX file. Rails detects the MIME type (supports .gpx, .gpx.gz, .gpx.bz2, .zip, .tar.gz, .tar.bz2, .tar), creates the Trace record with `inserted: false`, and uploads the file to S3.

2. **Queue**: Rails enqueues a `TraceImporterJob`. The priority is dynamic — the more pending traces a user has, the lower the priority. This prevents one user from blocking the queue.

3. **Import**: The job downloads the file from S3, parses it with the `gpx` gem, and inserts points into `gps_points` in batches of 1,000 using `activerecord-import`. Max points per trace: 1,000,000 (configurable).

4. **Images**: The job generates two GIF images (preview + thumbnail icon) and uploads them to S3.

5. **Done**: The job sets `inserted: true` and sends a success email. If parsing fails, it sends a failure email and deletes the trace.

### Infrastructure (Chef)

The job worker runs as a systemd service (`rails-jobs@traces`) configured in,https://github.com/openstreetmap/chef/blob/master/cookbooks/web/recipes/rails.rb. It runs on the existing web servers — no dedicated server needed anymore.

---

## 5. GPS Tile Rendering Pipeline (separate from the Rails app)

https://gps-tile.openstreetmap.org/

This pipeline is made of three C programs that run independently from the Rails app:

1. **[gpx-updater](https://github.com/openstreetmap/gpx-updater/)** — A Perl script that monitors the [public RSS feed](https://www.openstreetmap.org/traces/rss) of traces and downloads the GPX files for public traces.

2. **[gpx-import](https://github.com/e-n-f/gpx-import/)** (Erica Fisher's fork) — Extracts the trackpoints from the downloaded GPX files.

3. **[datamaps](https://github.com/e-n-f/datamaps/)** — Indexes the trackpoints and generates the raster tiles that editors display.

These are all C code with no active maintainer. The main problems:

- **Cannot delete traces.** If a user deletes or redacts a trace, the tile renderer cannot remove it from the tiles. In 2023, the team had to rebuild the entire tileset after deleting some traces — this took about a month. See [gpx-updater issue #4](https://github.com/openstreetmap/gpx-updater/issues/4).

- **No active maintenance.** These are old C programs. Finding someone to fix bugs or add features is difficult.

There is also **[planet-gpx-dump](https://github.com/openstreetmap/planet-gpx-dump)**, a Ruby script that generates a full dump of all public GPS traces. The dump at https://planet.openstreetmap.org/gps/ has not been updated since 2013 because the script needs to be updated to filter by permissions. also https://github.com/openstreetmap/operations/issues/258).

