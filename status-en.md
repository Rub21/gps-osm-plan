
# GPX Analysis in OpenStreetMap

Related repositories:

> - [`e-n-f/gpx-import`](https://github.com/e-n-f/gpx-import) (personal fork, archived)
> - [`openstreetmap/gpx-import`](https://github.com/openstreetmap/gpx-import) (official repository, archived Jul 2022)
> - [`openstreetmap/openstreetmap-website`](https://github.com/openstreetmap/openstreetmap-website) (current implementation)
> - [`openstreetmap/chef`](https://github.com/openstreetmap/chef) (infrastructure and configuration)


---
## 1. Context: What is gpx-import - https://github.com/openstreetmap/gpx-import?
`gpx-import` was the original **GPX Import Daemon** for OpenStreetMap, created in 2008. Its purpose was to asynchronously process GPX files uploaded by users to the platform, extracting trace points, generating preview images, and notifying the user via email.

- **Language:** C (94.5%), with Shell scripts and some C++ — https://github.com/openstreetmap/gpx-import/tree/master/src
- **License:** GNU GPL v2
- **Type:** System daemon (background process)
- **Current status:** Archived by the OpenStreetMap organization on **July 25, 2022** (read-only)
- **Integration issue:** [#1852 - Integrate the high-performance GPX importer](https://github.com/openstreetmap/openstreetmap-website/issues/1852)

The fork [`e-n-f/gpx-import`](https://github.com/e-n-f/gpx-import) was created by `Eric Fisher` and incorporated an experimental improvement: **calculation of the direction of movement (vector) for each point**. However, it received no further updates since 2013 and was never integrated into the official upstream.


---
## 2. Architecture until 2019

### 2.1 The C Daemon
> **Repo:** [`openstreetmap/gpx-import`](https://github.com/openstreetmap/gpx-import) (archived) | **Chef:** [`openstreetmap/chef`](https://github.com/openstreetmap/chef)

The daemon ran as an **independent systemd service**, with its own database user (`gpximport`) and direct access to PostgreSQL. Its configuration was managed through the Chef cookbook [`web::gpx`](https://github.com/openstreetmap/chef/blob/245c47e/cookbooks/web/recipes/gpx.rb) (removed in [`2a7d30a`](https://github.com/openstreetmap/chef/commit/2a7d30a), 10 Jun 2019).


**Key service environment variables for the old architecture:**
| Variable | Description |
|---|---|
| `GPX_SLEEP_TIME` | Wait time between processing cycles (40 s) |
| `GPX_PATH_TRACES` | Local path for GPX files (`/store/rails/gpx/traces`) |
| `GPX_PATH_IMAGES` | Local path for generated images |
| `GPX_PATH_TEMPLATES` | Email templates |
| `GPX_PGSQL_HOST/USER/PASS/DB` | Direct PostgreSQL connection |
| `GPX_LOG_FILE` | Log file |
| `GPX_MAIL_SENDER` | Notification sender |
**Chef recipe ([`cookbooks/web/recipes/gpx.rb`](https://github.com/openstreetmap/chef/blob/245c47e/cookbooks/web/recipes/gpx.rb)) — [commit history](https://github.com/openstreetmap/chef/commits/245c47e/cookbooks/web/recipes/gpx.rb):**


### 2.2 Dedicated server role
There was a specific Chef role called **[`web-gpximport`](https://github.com/openstreetmap/chef/blob/a7d96c8/roles/web-gpximport.rb)**, which means there was (or could have been) a server dedicated exclusively to running this daemon. This added operational complexity: an additional server to maintain, patch, and monitor.
### 2.3 Original flow diagram
```
User uploads GPX
      |
      v
Rails saves file to local disk (/store/rails/gpx/traces)
      |
      v
gpx-import daemon (C) — continuous polling with 40 s sleep
      |
      |-> Parses the GPX file (libexpat)
      |-> Inserts points into PostgreSQL (direct connection)
      |-> Generates preview image (libgd)
      |-> Updates metadata in DB
      '-> Sends email notification
```
---
## 3. The Migration (2019)

> **Key migration PRs in [`openstreetmap-website`](https://github.com/openstreetmap/openstreetmap-website)** — primary author: Andy Allan ([@gravitystorm](https://github.com/gravitystorm)), merges by Tom Hughes ([@tomhughes](https://github.com/tomhughes))

**1. [PR #2120](https://github.com/openstreetmap/openstreetmap-website/pull/2120) — Initial implementation with ActiveJob and gd-ffi** (28 Jan 2019)
   - Created `TraceImporterJob` and `TraceDestroyerJob` using Rails ActiveJob to process traces asynchronously, replacing the C daemon.
   - Introduced a configuration flag (`trace_use_job_queue`) for gradual activation without forcing immediate adoption.
   - Ported preview icon generation from the C daemon using the `gd2-ffij` gem (replacing an RMagick implementation that wasn't working).
   - Resolved issues [#281](https://github.com/openstreetmap/openstreetmap-website/issues/281) and [#1852](https://github.com/openstreetmap/openstreetmap-website/issues/1852).


**2. [PR #2131](https://github.com/openstreetmap/openstreetmap-website/pull/2131) — Bulk import with `activerecord-import`** (23 Mar 2019)
   - Replaced row-by-row tracepoint insertion with bulk inserts using the `activerecord-import` gem, achieving a **significant speedup even on SSDs**.
   - Implemented batch processing (batches of 1,000 points) to handle traces with millions of points without exhausting memory.

**3. [PR #2190](https://github.com/openstreetmap/openstreetmap-website/pull/2190) — Enabled by default** (28 Mar 2019)
   - Changed the `trace_use_job_queue` flag to `true` by default, activating the new system for all development environments.
   - In production, the change was not immediate: the Chef configuration continued to override the value. The actual production deployment was coordinated separately.


**4. [PR #2204](https://github.com/openstreetmap/openstreetmap-website/pull/2204) — Animated images for traces** (10 Jun 2019)
   - Added generation of **animated GIFs** showing the trace route point by point, based on previous work by @mmd-osm.
   - Included a workaround for a libgd bug that caused segfaults.
   - Waited for the `gd2-ffij` gem v0.4.0 to include official animation support before merging, avoiding dependencies on forks.

**5. [PR #2422](https://github.com/openstreetmap/openstreetmap-website/pull/2422) — Removal of references to the external daemon** (1 Nov 2019)
   - Cleaned up the code by removing the last references to the external C daemon `gpx-import`, formally closing the migration cycle in the website codebase.

### 3.1 Removal commit
On **June 10, 2019**, Tom Hughes made the commit:
> **"Remove gpximport role and recipe"** — https://github.com/openstreetmap/chef/commit/2a7d30a
This commit removed 101 lines of infrastructure code:
- `cookbooks/web/recipes/gpx.rb` (94 lines)
- `roles/web-gpximport.rb` (7 lines)
This marked the end of the C daemon as an active component of the production infrastructure. The official repository [`openstreetmap/gpx-import`](https://github.com/openstreetmap/gpx-import) was formally archived years later, in July 2022.

### 3.2 Migration motivations
| Factor | Previous problem | Adopted solution |
|---|---|---|
| **Infrastructure complexity** | Dedicated server + external daemon | Job integrated into Rails |
| **Language** | C required compilation in production | Native Ruby in the app |
| **Storage** | Local file system (single point of failure) | AWS S3 (scalable, redundant) |
| **DB security** | `gpximport` user with direct PostgreSQL access | Access through Rails ORM |
| **Maintenance** | System C dependencies (libexpat, libgd, etc.) | Ruby gem `gpx` |
| **Observability** | Separate log file | Integrated into the Rails stack |


---
## 4. Current Architecture

### 4.1 Technology stack
GPX processing is now part of **[`openstreetmap-website`](https://github.com/openstreetmap/openstreetmap-website)**, the Rails application that powers all of openstreetmap.org. The key components are:
- **[`app/controllers/traces_controller.rb`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/traces_controller.rb)** — Handles trace upload and management
- **[`app/models/trace.rb`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/models/trace.rb)** — Business logic, parsing, and storage
- **[`app/jobs/trace_importer_job.rb`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/jobs/trace_importer_job.rb)** — Asynchronous import job ([PR #2120](https://github.com/openstreetmap/openstreetmap-website/pull/2120))
- **[`app/jobs/trace_destroyer_job.rb`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/jobs/trace_destroyer_job.rb)** — Asynchronous deletion job
- **[`gpx`](https://github.com/dougfales/gpx) gem** — GPX format parsing in Ruby
- **ActiveJob** — Rails job queue framework
- **AWS S3** — Storage for GPX files and images
### 4.2 Data model (`Trace`)
The underlying table is still called `gpx_files` (historical compatibility):
```ruby
# Table name: gpx_files
#
#  id          :bigint           not null, primary key
#  user_id     :bigint           not null
#  visible     :boolean          default(TRUE)
#  name        :string           default("")
#  size        :bigint
#  latitude    :float
#  longitude   :float
#  timestamp   :datetime         not null
#  description :string           default("")
#  inserted    :boolean          not null   # false = pending import
#  visibility  :enum             default("public")
#                                # values: private | public | trackable | identifiable
```
**Relationships:**
- `has_many :tags` (table `gpx_file_tags`)
- `has_many :points` (table `tracepoints`)
- `has_one_attached :file` -> S3 bucket `openstreetmap-gps-traces`
- `has_one_attached :image` -> S3 bucket `openstreetmap-gps-images`
- `has_one_attached :icon` -> S3 bucket `openstreetmap-gps-images`
**Available visibilities:**
| Value | Description |
|---|---|
| `private` | Only visible to the owner |
| `public` | Publicly visible, without user identification |
| `trackable` | Public with timestamps, without username |
| `identifiable` | Public with timestamps and username |
### 4.3 Complete technical flow
#### Step 1 — File upload ([`TracesController#create`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/traces_controller.rb))
```ruby
def create
  @trace = do_create(params[:trace][:gpx_file],
                     params[:trace][:tagstring],
                     params[:trace][:description],
                     params[:trace][:visibility])
  if @trace.id
    flash[:warning] = t(".traces_waiting", :count => ...) if pending_count > 4
    @trace.schedule_import   # <- enqueues the job
    redirect_to :action => :index
  end
end
```
**The `do_create` method:**
1. Sanitizes the filename (replaces non-alphanumeric characters with `_`)
2. Detects the real MIME type using `/usr/bin/file` (does not trust the extension)
3. Creates the `Trace` record with `inserted: false` (marks as pending)
4. Uploads the file to S3 via ActiveStorage
**Supported file types:**
| MIME type | Logical extension |
|---|---|
| `application/gpx+xml` | `.gpx` |
| `application/gzip` | `.gpx.gz` |
| `application/x-bzip2` | `.gpx.bz2` |
| `application/zip` | `.zip` |
| `application/x-tar+gzip` | `.tar.gz` |
| `application/x-tar+x-bzip2` | `.tar.bz2` |
| `application/x-tar` | `.tar` |
#### Step 2 — Job enqueuing ([`Trace#schedule_import`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/models/trace.rb))
```ruby
def schedule_import
  TraceImporterJob.new(self).enqueue(
    :priority => user.traces.where(:inserted => false).count
  )
end
```
The **priority is dynamic**: the more pending traces a user has, the higher the number (lower priority), preventing a single user from saturating the queue.
#### Step 3 — Job execution ([`TraceImporterJob#perform`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/jobs/trace_importer_job.rb))
```ruby
class TraceImporterJob < ApplicationJob
  queue_as :traces
  def perform(trace)
    gpx = trace.import
    if gpx.actual_points.positive?
      UserMailer.gpx_success(trace, gpx.actual_points).deliver
    else
      UserMailer.gpx_failure(trace, "0 points parsed ok...").deliver
      trace.destroy
    end
  rescue LibXML::XML::Error => e
    UserMailer.gpx_failure(trace, e).deliver
    trace.destroy
  rescue StandardError => e
    UserMailer.gpx_failure(trace, "#{e}\n#{e.backtrace.join("\n")}").deliver
    trace.destroy
  end
end
```
The job handles three scenarios:
- **Success:** notifies the user with the count of imported points
- **XML parsing failure:** notifies and deletes the trace
- **Unexpected error:** notifies with stack trace and deletes the trace
#### Step 4 — Actual import ([`Trace#import`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/models/trace.rb))
```ruby
def import
  file.open do |file|
    gpx = GPX::File.new(file.path, :maximum_points => Settings.max_trace_size)
    # 1. Delete existing points (safe re-import)
    Tracepoint.where(:trace => id).delete_all
    # 2. Import points in batches of 1,000
    gpx.points.each_slice(1_000) do |points|
      tracepoints = points.map do |point|
        Tracepoint.new(
          lat:       point.latitude,
          lon:       point.longitude,
          altitude:  point.altitude,
          timestamp: point.timestamp,
          gpx_id:    id,
          trackid:   point.segment   # <- preserves trace segments
        )
      end
      Tracepoint.import!(tracepoints)  # bulk insert with activerecord-import (PR #2131)
    end
    # 3. Calculate bounding box and generate images
    if gpx.actual_points.positive?
      # ... min/max lat/lon calculation from DB ...
      image.attach(:io => gpx.picture(min_lat, min_lon, max_lat, max_lon, actual_points),
                   :filename => "#{id}.gif", :content_type => "image/gif")
      icon.attach(:io => gpx.icon(min_lat, min_lon, max_lat, max_lon),
                  :filename => "#{id}_icon.gif", :content_type => "image/gif")
      self.size    = gpx.actual_points
      self.inserted = true   # <- marks as processed
      save!
    end
  end
end
```
**Notable aspects:**
- Configurable point limit per trace (`Settings.max_trace_size`)
- Insertion in batches of 1,000 points for efficiency
- Preserves the segment number (`trackid`) of each point
- Generates two images: a normal-sized one and a thumbnail icon (both GIF)
- Saves the position of the **first point** (`latitude`, `longitude`) for quick geolocation
### 4.4 Current infrastructure (Chef)
The Rails job worker is configured as a **parameterized systemd service** in [`cookbooks/web/recipes/rails.rb`](https://github.com/openstreetmap/chef/blob/master/cookbooks/web/recipes/rails.rb):
```ruby
systemd_service "rails-jobs@" do
  description "Rails job queue runner"
  type        "simple"
  environment "RAILS_ENV"   => "production",
              "QUEUE"       => "%I",       # <- parameter: queue name
              "SLEEP_DELAY" => "60",
              "SECRET_KEY_BASE" => web_passwords["secret_key_base"]
  user              "rails"
  working_directory rails_directory
  exec_start        "#{node[:ruby][:bundle]} exec rails jobs:work"
  restart           "on-failure"
  nice              10        # lower CPU priority
  sandbox           :enable_network => true
  memory_deny_write_execute false
  read_write_paths  "/var/log/web"
end
```
The `rails-jobs@traces` service specifically processes the GPX import queue. The `%I` parameter in systemd is replaced by the queue name when instantiating the service, allowing multiple workers for different queues.
**S3 Storage:**
| Bucket | Content | ACL |
|---|---|---|
| `openstreetmap-gps-traces` | Original GPX files | `public-read` |
| `openstreetmap-gps-images` | Images and thumbnails | `public-read` |
Region: `eu-west-1` (Ireland), with `use_dualstack_endpoint: true` (IPv4 + IPv6).
---
## 5. Comparative Architecture Diagram
```
================================================================
  PREVIOUS ARCHITECTURE (until 2019)
================================================================
  [User]
      | HTTP upload
      v
  [Rails App]
      | Saves to /store/rails/gpx/traces (local disk)
      | Marks trace as inserted=false in DB
      v
  [gpx-import daemon — separate C process]
      | Polling every 40 seconds
      | Reads files from local disk
      | Direct PostgreSQL connection (gpximport user)
      | Generates images -> local disk
      v
  [PostgreSQL] <- direct access without ORM
================================================================
  CURRENT ARCHITECTURE (since 2019)
================================================================
  [User]
      | HTTP upload
      v
  [Rails App — TracesController]
      | Uploads file to S3 (ActiveStorage)
      | Creates Trace record (inserted=false)
      | Enqueues TraceImporterJob (dynamic priority)
      v
  [ActiveJob Worker — rails-jobs@traces]
      | Downloads file from S3
      | Parses with Ruby 'gpx' gem
      | Inserts Tracepoints in bulk (batches of 1,000)
      | Generates GIF images -> uploads to S3
      | Updates Trace (inserted=true)
      v
  [PostgreSQL] <- through Rails ORM (ActiveRecord)
  [AWS S3]     <- GPX files + images
```
---
## 6. Final Comparative Table
| Aspect | Previous architecture | Current architecture |
|---|---|---|
| **Implementation** | [External C daemon](https://github.com/openstreetmap/gpx-import) | ActiveJob in Rails ([PR #2120](https://github.com/openstreetmap/openstreetmap-website/pull/2120)) |
| **GPX Parsing** | [libexpat](https://github.com/openstreetmap/gpx-import/blob/master/src/main.c) (C) | [`gpx`](https://github.com/dougfales/gpx) gem (Ruby) |
| **Activation** | Polling every 40 s | Job enqueued at upload time |
| **Queue priority** | Did not exist | Dynamic based on user's pending traces |
| **File storage** | Local file system | AWS S3 (eu-west-1) — [Chef config](https://github.com/openstreetmap/chef/blob/master/cookbooks/web/recipes/rails.rb) |
| **Image storage** | Local file system | AWS S3 (eu-west-1) — [Chef config](https://github.com/openstreetmap/chef/blob/master/cookbooks/web/recipes/rails.rb) |
| **DB access** | Direct connection (`gpximport` user) | ActiveRecord ORM (`rails` user) |
| **Dedicated server** | Yes (role [`web-gpximport`](https://github.com/openstreetmap/chef/blob/a7d96c8/roles/web-gpximport.rb)) | No (runs on existing web servers) |
| **Dependency management** | `apt` + compilation in production | Bundler (Gemfile) |
| **Error handling** | Log file + email | Ruby exceptions + email + trace destruction |
| **Point limit** | Hardcoded in C | Configurable (`Settings.max_trace_size`) |
| **Supported formats** | GPX, gz, bz2, zip, tar | GPX, gz, bz2, zip, tar.gz, tar.bz2 |
| **Point insertion** | Row by row | Bulk insert in batches of 1,000 ([PR #2131](https://github.com/openstreetmap/openstreetmap-website/pull/2131)) |
| **Observability** | Separate log file | Integrated Rails stack |
---
## 7. Relevance to the Proposal
This research demonstrates that OpenStreetMap followed a path of **progressive architectural simplification**: it eliminated a specialized C component that required dedicated infrastructure, and integrated it into the Rails monolith by leveraging its native mechanisms (ActiveJob, ActiveStorage).
The key learning points to consider in the proposal are:
1. **Integration is preferable to premature specialization.** An external C daemon is only justified when Ruby cannot handle the load. In this case, Ruby with bulk inserts proved sufficient.
2. **ActiveJob decouples processing without adding infrastructure.** The same worker that processes other Rails queues can process GPX, reducing the operational surface.
3. **S3 as a decoupled storage layer** eliminates local disk dependencies and facilitates horizontal scaling of application servers.
4. **Dynamic queue priority** is a simple but effective mechanism to prevent abuse (a user uploading thousands of files does not block others).
5. **The [`e-n-f/gpx-import`](https://github.com/e-n-f/gpx-import) fork** with its addition of direction vectors was never integrated, which suggests that this functionality was not considered a priority by the OSM community at the time.
