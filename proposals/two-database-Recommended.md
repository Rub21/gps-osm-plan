## Move GPS tables to a separate database - Recommended 

**Status:** Idea — mentioned by Pablo and Grant

---

The idea is to move the 3 GPS tables to a second PostgreSQL database, separate from the main OSM database. I think this could help with maintenance — for example, reindexing or running migrations on GPS data without touching the rest of the site.

### GPS tables that would move

```
gps_points      ← 31.8B rows
gpx_files       ← trace metadata (name, visibility, user_id...)
gpx_file_tags   ← tags per trace
```

`gpx_files.user_id` would stay as a plain reference value — no FK constraint, just a number.

---

### How user_id is used in the app

I checked the code and most of the user_id usage is not a problem — the app already has `current_user.id` available, so no cross-DB join is needed. Only one case needs a small fix:

| Use | Code | Cross-DB problem? |
|---|---|---|
| List user traces | [traces_controller.rb:55](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/traces_controller.rb#L55) | No — `current_user.id` is already available |
| Check ownership | [traces_controller.rb:75](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/traces_controller.rb#L75), [api/traces_controller.rb:18](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/api/traces_controller.rb#L18) | No — change to compare IDs |
| Create trace | [traces_controller.rb:128](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/traces_controller.rb#L128), [api/traces_controller.rb:96](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/api/traces_controller.rb#L96) | No — just saves the ID |
| `trace.user.display_name` | [traces_controller.rb:163](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/traces_controller.rb#L163) | Yes — need extra work to do here |


`User` also has `has_many :traces` and a `traces_count` counter cache — those would need to be replaced with manual queries to the GPS DB.

---

### What Pablo and Grant said

- **Pablo**: keep it in the same database for now. A separate database adds maintenance complexity. Leave for later.
- **Grant**: not against it, but does not know how well Rails handles two databases — "that's more a question for Andy and Tom".

---

### Rails support

I checked and Rails 8.1 (used by this project) supports multiple databases via `connects_to` — available since Rails 6. It is not configured yet but it is possible:

```ruby
# trace.rb and tracepoint.rb
connects_to database: { writing: :gps_db, reading: :gps_db }
```

---

### Things that would need to change

- `has_many :traces` in `User` — replace with manual query
- `traces_count` counter cache in `users` table — update manually or count on-demand
- No DB-level FK constraints — integrity handled in app code
- Cascade deletes (`dependent: :delete_all`) — handle in code, not PostgreSQL
