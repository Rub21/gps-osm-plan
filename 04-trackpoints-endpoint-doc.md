# GET /api/0.6/trackpoints - How it works

Documentation of the current trackpoints endpoint for the GPS traces discussion with Pablo.

---

## 1. Request

```
GET /api/0.6/trackpoints?bbox=min_lon,min_lat,max_lon,max_lat&page=0
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `bbox`    | Yes      | Bounding box: `min_lon,min_lat,max_lon,max_lat`. Max area: 0.25 sq degrees |
| `page`    | No       | Page number (0-indexed). Default: 0 |

Returns **5000 points per page**.

Response: GPX 1.0 XML file, as attachment (`files/tracks.gpx`).

e.g for bbox: `169.1399141,-44.6967154,169.2056708,-44.6435629`

```sh
GET https://api.openstreetmap.org/api/0.6/trackpoints?bbox=169.1399141,-44.6967154,169.2056708,-44.6435629&page=0
GET https://api.openstreetmap.org/api/0.6/trackpoints?bbox=169.1399141,-44.6967154,169.2056708,-44.6435629&page=1
GET https://api.openstreetmap.org/api/0.6/trackpoints?bbox=169.1399141,-44.6967154,169.2056708,-44.6435629&page=2
GET https://api.openstreetmap.org/api/0.6/trackpoints?bbox=169.1399141,-44.6967154,169.2056708,-44.6435629&page=3
GET https://api.openstreetmap.org/api/0.6/trackpoints?bbox=169.1399141,-44.6967154,169.2056708,-44.6435629&page=4
GET https://api.openstreetmap.org/api/0.6/trackpoints?bbox=169.1399141,-44.6967154,169.2056708,-44.6435629&page=5
GET https://api.openstreetmap.org/api/0.6/trackpoints?bbox=169.1399141,-44.6967154,169.2056708,-44.6435629&page=6
GET https://api.openstreetmap.org/api/0.6/trackpoints?bbox=169.1399141,-44.6967154,169.2056708,-44.6435629&page=7
GET https://api.openstreetmap.org/api/0.6/trackpoints?bbox=169.1399141,-44.6967154,169.2056708,-44.6435629&page=8
GET https://api.openstreetmap.org/api/0.6/trackpoints?bbox=169.1399141,-44.6967154,169.2056708,-44.6435629&page=9
GET https://api.openstreetmap.org/api/0.6/trackpoints?bbox=169.1399141,-44.6967154,169.2056708,-44.6435629&page=10
GET https://api.openstreetmap.org/api/0.6/trackpoints?bbox=169.1399141,-44.6967154,169.2056708,-44.6435629&page=11
GET https://api.openstreetmap.org/api/0.6/trackpoints?bbox=169.1399141,-44.6967154,169.2056708,-44.6435629&page=12
```
---

## 2. Code flow

### Route
The URL is `/api/0.6/trackpoints` but the controller se llama `TracepointsController`. Es una inconsistencia parece que es historico en  JOSM e iD ya usan esta URL.

-> [config/routes.rb:90](https://github.com/openstreetmap/openstreetmap-website/blob/master/config/routes.rb#L90)

### Controller
El controller recibe el `page` number (0-indexed), calcula el OFFSET (`page * 5000`), valida el bounding box, y hace **dos queries separadas** que combina con `UNION ALL`:

- **Query 1** (trackable/identifiable): Trazas donde el usuario permite que se vean agrupadas por traza. Se ordenan por `gpx_id DESC, trackid ASC, timestamp ASC`.
- **Query 2** (public/private): Trazas anonimizadas. Los puntos se mezclan y se ordenan por `latitude, longitude, timestamp` — no se puede saber a que traza pertenece cada punto.

Despues aplica `OFFSET` y `LIMIT 5000` sobre el resultado combinado.

-> [app/controllers/api/tracepoints_controller.rb](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/api/tracepoints_controller.rb)

### Que datos trae el query vs que se devuelve al cliente

El query trae todas las columnas de `gps_points` (latitude, longitude, altitude, trackid, gpx_id, timestamp, tile). Pero la vista GPX **no devuelve todo**: ejemplo en l a

| Dato | identifiable | trackable | public/private |
|------|-------------|-----------|----------------|
| lat/lon | Si | Si | Si |
| timestamp | Si | Si | No |
| altitude | No | No | No |
| name/desc/url | Si | No | No |
| trackid (agrupacion por segmento) | Si | Si | No (todo mezclado) |

**altitude nunca se devuelve** en la respuesta GPX aunque esta en la base de datos. La vista solo usa `lat`, `lon`, y condicionalmente `timestamp`. La altitud se ignora completamente.

-> [app/views/api/tracepoints/index.gpx.builder](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/views/api/tracepoints/index.gpx.builder)

### SQL generado (aproximado)

El query final es algo asi:

1. Filtra puntos dentro del bounding box usando **QuadTile index** (columna `tile`) + chequeo de rango de `latitude`/`longitude`
2. Hace JOIN con `gpx_files` para verificar la visibilidad
3. Combina ambos resultados con UNION ALL
4. Aplica OFFSET y LIMIT

Uno de los problemas es El OFFSET es el problema principal: PostgreSQL tiene que **escanear y descartar** todas las filas antes del offset.

---

## 3. Database schema (gps_points)

La tabla `gps_points` tiene aproximadamente **31.8 billones de filas**

Columnas:
- **altitude** (float) — opcional, puede ser NULL
- **trackid** (integer) — ID del segmento dentro del archivo GPX
- **latitude** (integer) — WGS84 multiplicado por 10,000,000 (ejemplo: -12.0464 se guarda como -120464000)
- **longitude** (integer) — WGS84 multiplicado por 10,000,000
- **gpx_id** (bigint) — FK a `gpx_files.id`
- **timestamp** (datetime) — cuando se capturo el punto
- **tile** (bigint) — indice QuadTile para filtrado espacial

**La tabla NO tiene primary key (no hay columna `id`).** https://github.com/openstreetmap/openstreetmap-website/blob/master/db/migrate/001_create_osm_db.rb#L75 

Indices:
- `points_gpxid_idx` en `gpx_id`
- `points_tile_idx` en `tile`

El modelo convierte latitude/longitude de integers a floats dividiendo por 10,000,000.

-> [app/models/tracepoint.rb](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/models/tracepoint.rb)
-> [app/models/concerns/geo_record.rb](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/models/concerns/geo_record.rb) (modulo compartido: convierte lat/lon integers a floats, define el scope `bbox` para filtrado espacial, y calcula el QuadTile antes de guardar)
-> [db/migrate/001_create_osm_db.rb](https://github.com/openstreetmap/openstreetmap-website/blob/master/db/migrate/001_create_osm_db.rb) (creacion de la tabla)

---

## 4. Spatial filtering: QuadTile + range check

El filtrado espacial usa dos pasos:

1. **QuadTile**: Un indice espacial basado en una "space-filling curve" (parecido a un Z-order curve). El gem `quad_tile` genera los tile IDs para un bounding box, y Postgres filtra rapido usando el indice `points_tile_idx`. Esto reduce drasticamente los candidatos.

2. **Range check**: Despues del filtro de tiles, se verifica que `latitude` y `longitude` esten dentro del rango exacto del bbox. Esto es necesario porque los tiles son cuadrados aproximados y pueden incluir puntos fuera del bbox.


-> [lib/osm.rb:522-528](https://github.com/openstreetmap/openstreetmap-website/blob/master/lib/osm.rb#L522-L528) (sql_for_area)
-> [app/models/concerns/geo_record.rb:25](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/models/concerns/geo_record.rb#L25) (bbox scope)

---

## 5. Visibility and ordering

| Visibility     | Quien lo ve          | Orden en la respuesta                     | Muestra gpx_files.name | Muestra timestamp? |
|----------------|----------------------|-------------------------------------------|------------------------|--------------------|
| `identifiable` | Todos                | Agrupado por gpx_id, trackid, timestamp   | Si                     | Si                 |
| `trackable`    | Todos                | Agrupado por gpx_id, trackid, timestamp   | No                     | Si                 |
| `public`       | Todos                | Mezclado por lat, lon, timestamp          | No                     | No                 |
| `private`      | Todos (dentro de bbox)| Mezclado por lat, lon, timestamp         | No                     | No                 |

Los puntos de trazas `trackable`/`identifiable` se agrupan por traza y segmento en la respuesta. Los puntos de trazas `public`/`private` se mezclan todos juntos en un unico track anonimo — es imposible saber que puntos pertenecen a que traza.


---

## 6. View (GPX XML response)

La vista itera sobre los puntos y los agrupa en elementos `<trk>` (track) y `<trkseg>` (track segment).

Ejemplo de respuesta:

- **Traza identifiable**: Incluye `<name>`, `<desc>`, `<url>` del trace, y cada punto tiene `<time>` con el timestamp.
- **Traza trackable**: No incluye nombre/descripcion/url, pero los puntos si tienen `<time>`.
- **Traza public/private**: Un solo track anonimo sin nombre, sin timestamps. Solo lat/lon.

El formato es GPX 1.0. Los puntos se devuelven como `<trkpt>` con atributos `lat` y `lon` en grados decimales (7 decimales de precision).

-> [app/views/api/tracepoints/index.gpx.builder](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/views/api/tracepoints/index.gpx.builder)

---

## 7. The pagination problem (OFFSET)

### Comportamiento actual

- page=0: OFFSET 0, LIMIT 5000 — rapido
- page=1: OFFSET 5000, LIMIT 5000 — ok
- page=10: OFFSET 50000, LIMIT 5000 — empieza a ser lento
- page=100: OFFSET 500000, LIMIT 5000 — muy lento
- page=1000: OFFSET 5,000,000, LIMIT 5000 — timeout

PostgreSQL tiene que **escanear y descartar** todas las filas antes del offset. Con 31.8 billones de filas y un indice QuadTile que puede devolver millones de candidatos en zonas densas (Berlin, Japon), las paginas altas causan timeouts.

### Lo que ya se arreglo (PR #4089, 2023)

En 2023 el endpoint `/api/0.6/gpx_files` (listar trazas) tenia el mismo problema con OFFSET. Lo arreglaron cambiando a keyset pagination: en vez de `?page=0, ?page=1, ?page=2...` ahora el cliente manda el ID de la ultima traza que vio:

- Antes: `GET /api/0.6/gpx_files?page=50` — lento, Postgres salta 5000 filas
- Despues: `GET /api/0.6/gpx_files?before=8500` — rapido, Postgres va directo al id < 8500 usando el indice

Siempre es igual de rapido sin importar que tan "adelante" estes en los resultados.

El endpoint `/api/0.6/trackpoints` **no fue migrado** y sigue usando OFFSET con `?page=N`. se podria implementar la misma solucion?

- PR de migracion 2023: https://github.com/openstreetmap/openstreetmap-website/pull/4089

### Propuesta: keyset pagination

En vez de OFFSET, el cliente envia los ultimos valores que vio (por ejemplo, el ultimo `gpx_id`, `trackid`, `timestamp`) y Postgres salta directamente a esa posicion usando indices. No necesita escanear/descartar filas.

Esto **no son cursores de base de datos** (como `pg_fetch_row` en PHP). No se persiste nada en el servidor entre requests. El cliente simplemente manda el ultimo valor visto como parametro, y el server hace un `WHERE ... > last_value` en vez de `OFFSET`.

**Desafio 1:** La query de `public`/`private` ordena por `(latitude, longitude, timestamp)` que no tiene un indice unico. Necesita mas analisis.

**Desafio 2:** La tabla NO tiene primary key (no hay columna `id`), lo que hace keyset pagination menos directo que en otras tablas.

---

## 8. Other observations

- **No PostGIS**: El codigo actual usa QuadTile (gem custom) para queries espaciales. PostGIS se esta introduciendo por primera vez via el PR de Pablo para notas anonimas (#6713).
- **JOSM**: JOSM llama a este endpoint en background y silenciosamente ignora errores/timeouts. Los usuarios no ven los fallos.
- **Max bbox size**: 0.25 grados cuadrados.
- **API timeout**: 300 segundos (5 minutos).
- **No hay rate limiting** especifico para este endpoint, solo el timeout general del API.

---

## 9. Files involved

| File | What it does |
|------|-------------|
| [config/routes.rb:90](https://github.com/openstreetmap/openstreetmap-website/blob/master/config/routes.rb#L90) | Route definition |
| [app/controllers/api/tracepoints_controller.rb](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/api/tracepoints_controller.rb) | Controller con OFFSET pagination |
| [app/models/tracepoint.rb](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/models/tracepoint.rb) | Model que mapea a la tabla `gps_points` |
| [app/models/trace.rb](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/models/trace.rb) | Model del GPX file (tabla `gpx_files`) |
| [app/models/concerns/geo_record.rb](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/models/concerns/geo_record.rb) | SCALE constant, bbox scope, conversion lat/lon |
| [app/views/api/tracepoints/index.gpx.builder](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/views/api/tracepoints/index.gpx.builder) | Template GPX 1.0 XML |
| [lib/osm.rb:522-528](https://github.com/openstreetmap/openstreetmap-website/blob/master/lib/osm.rb#L522-L528) | sql_for_area() - QuadTile + range check |
| [lib/bounding_box.rb](https://github.com/openstreetmap/openstreetmap-website/blob/master/lib/bounding_box.rb) | BoundingBox validation |
| [config/settings.yml:33](https://github.com/openstreetmap/openstreetmap-website/blob/master/config/settings.yml#L33) | tracepoints_per_page: 5000 |
