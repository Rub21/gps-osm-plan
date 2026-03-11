
# Análisis de GPX en OpenStreetMap

Related repositories: 

> - [`e-n-f/gpx-import`](https://github.com/e-n-f/gpx-import) (fork personal, archivado)
> - [`openstreetmap/gpx-import`](https://github.com/openstreetmap/gpx-import) (repositorio oficial, archivado Jul 2022)
> - [`openstreetmap/openstreetmap-website`](https://github.com/openstreetmap/openstreetmap-website) (implementación actual)
> - [`openstreetmap/chef`](https://github.com/openstreetmap/chef) (infraestructura y configuración)


---
## 1. Contexto: ¿Qué es gpx-import - https://github.com/openstreetmap/gpx-import?
`gpx-import` fue el **GPX Import Daemon** original de OpenStreetMap, creado en 2008. Su propósito era procesar de forma asíncrona los archivos GPX que los usuarios subían a la plataforma, extrayendo los puntos de traza, generando imágenes de previsualización y notificando al usuario por correo electrónico.

- **Lenguaje:** C (94.5%), con scripts Shell y algo de C++ — https://github.com/openstreetmap/gpx-import/tree/master/src
- **Licencia:** GNU GPL v2
- **Tipo:** Daemon de sistema (proceso en segundo plano)
- **Estado actual:** Archivado por la organización OpenStreetMap el **25 de julio de 2022** (read-only)
- **Issue de integración:** [#1852 - Integrate the high-performance GPX importer](https://github.com/openstreetmap/openstreetmap-website/issues/1852)

El fork [`e-n-f/gpx-import`](https://github.com/e-n-f/gpx-import) fue creado por `Eric Fisher` e incorporó una mejora experimental: **cálculo de la dirección de movimiento (vector) para cada punto**. Sin embargo, no recibió más actualizaciones desde el 2013 y tampoco fue integrado al upstream oficial.


---
## 2. Arquitectura hasta 2019

### 2.1 El Daemon en C
> **Repo:** [`openstreetmap/gpx-import`](https://github.com/openstreetmap/gpx-import) (archivado) | **Chef:** [`openstreetmap/chef`](https://github.com/openstreetmap/chef)

El daemon corría como un **servicio systemd independiente**, con su propio usuario de base de datos (`gpximport`) y acceso directo a PostgreSQL. Su configuración se gestionaba a través del cookbook Chef [`web::gpx`](https://github.com/openstreetmap/chef/blob/245c47e/cookbooks/web/recipes/gpx.rb) (eliminado en [`2a7d30a`](https://github.com/openstreetmap/chef/commit/2a7d30a), 10 Jun 2019).


**Variables de entorno clave del servicio para la arquitectura antigua:**
| Variable | Descripción |
|---|---|
| `GPX_SLEEP_TIME` | Tiempo de espera entre ciclos de procesamiento (40 s) |
| `GPX_PATH_TRACES` | Ruta local de archivos GPX (`/store/rails/gpx/traces`) |
| `GPX_PATH_IMAGES` | Ruta local de imágenes generadas |
| `GPX_PATH_TEMPLATES` | Plantillas de correo electrónico |
| `GPX_PGSQL_HOST/USER/PASS/DB` | Conexión directa a PostgreSQL |
| `GPX_LOG_FILE` | Archivo de log |
| `GPX_MAIL_SENDER` | Remitente de notificaciones |
**Receta Chef ([`cookbooks/web/recipes/gpx.rb`](https://github.com/openstreetmap/chef/blob/245c47e/cookbooks/web/recipes/gpx.rb)) — [historial de commits](https://github.com/openstreetmap/chef/commits/245c47e/cookbooks/web/recipes/gpx.rb):**


### 2.2 Rol de servidor dedicado
Existía un rol Chef específico llamado **[`web-gpximport`](https://github.com/openstreetmap/chef/blob/a7d96c8/roles/web-gpximport.rb)**, lo que significa que había (o podía haber) un servidor dedicado exclusivamente a ejecutar este daemon. Esto añadía complejidad operativa: un servidor adicional que mantener, parchear y monitorear.
### 2.3 Diagrama de flujo original
```
Usuario sube GPX
      │
      ▼
Rails guarda archivo en disco local (/store/rails/gpx/traces)
      │
      ▼
gpx-import daemon (C) — polling continuo con sleep de 40 s
      │
      ├─► Parsea el archivo GPX (libexpat)
      ├─► Inserta puntos en PostgreSQL (conexión directa)
      ├─► Genera imagen de previsualización (libgd)
      ├─► Actualiza metadatos en DB
      └─► Envía notificación por correo
```
---
## 3. La Migración (2019)

> **PRs clave de la migración en [`openstreetmap-website`](https://github.com/openstreetmap/openstreetmap-website)** — autor principal: Andy Allan ([@gravitystorm](https://github.com/gravitystorm)), merges por Tom Hughes ([@tomhughes](https://github.com/tomhughes))

**1. [PR #2120](https://github.com/openstreetmap/openstreetmap-website/pull/2120) — Implementación inicial con ActiveJob y gd-ffi** (28 Ene 2019)
   - Creó `TraceImporterJob` y `TraceDestroyerJob` usando Rails ActiveJob para procesar trazas de forma asíncrona, reemplazando al daemon C.
   - Introdujo un flag de configuración (`trace_use_job_queue`) para activación gradual sin forzar adopción inmediata.
   - Portó la generación de iconos de previsualización del daemon C usando la gema `gd2-ffij` (reemplazando una implementación RMagick que no funcionaba).
   - Resolvió los issues [#281](https://github.com/openstreetmap/openstreetmap-website/issues/281) y [#1852](https://github.com/openstreetmap/openstreetmap-website/issues/1852).


**2. [PR #2131](https://github.com/openstreetmap/openstreetmap-website/pull/2131) — Bulk import con `activerecord-import`** (23 Mar 2019)
   - Reemplazó la inserción fila-por-fila de tracepoints por bulk inserts usando la gema `activerecord-import`, logrando un **speedup significativo incluso en SSDs**.
   - Implementó procesamiento en batches (lotes de 1.000 puntos) para manejar trazas con millones de puntos sin agotar la memoria.

**3. [PR #2190](https://github.com/openstreetmap/openstreetmap-website/pull/2190) — Habilitado por defecto** (28 Mar 2019)
   - Cambió el flag `trace_use_job_queue` a `true` por defecto, activando el nuevo sistema para todos los entornos de desarrollo.
   - En producción el cambio no fue inmediato: la configuración Chef seguía sobreescribiendo el valor. El despliegue real en producción se coordinó por separado.


**4. [PR #2204](https://github.com/openstreetmap/openstreetmap-website/pull/2204) — Imágenes animadas para traces** (10 Jun 2019)
   - Añadió generación de **GIFs animados** que muestran el recorrido de la traza punto a punto, basado en trabajo previo de @mmd-osm.
   - Incluía un workaround para un bug en libgd que causaba segfaults.
   - Esperó a que la gema `gd2-ffij` v0.4.0 incluyera soporte oficial de animación antes de hacer merge, evitando dependencias en forks.

**5. [PR #2422](https://github.com/openstreetmap/openstreetmap-website/pull/2422) — Eliminación de referencias al daemon externo** (1 Nov 2019)
   - Limpió el código eliminando las últimas referencias al daemon C externo `gpx-import`, cerrando formalmente el ciclo de migración en el codebase del website.

### 3.1 Commit de eliminación
El **10 de junio de 2019**, Tom Hughes realizó el commit:
> **"Remove gpximport role and recipe"** — https://github.com/openstreetmap/chef/commit/2a7d30a
Este commit eliminó 101 líneas de código de infraestructura:
- `cookbooks/web/recipes/gpx.rb` (94 líneas)
- `roles/web-gpximport.rb` (7 líneas)
Esto marcó el fin del daemon C como componente activo de la infraestructura de producción. El repositorio oficial [`openstreetmap/gpx-import`](https://github.com/openstreetmap/gpx-import) fue formalmente archivado años después, en julio de 2022.

### 3.2 Motivaciones de la migración
| Factor | Problema anterior | Solución adoptada |
|---|---|---|
| **Complejidad de infra** | Servidor dedicado + daemon externo | Job integrado en Rails |
| **Lenguaje** | C requería compilación en producción | Ruby nativo en la app |
| **Almacenamiento** | Sistema de archivos local (single point of failure) | AWS S3 (escalable, redundante) |
| **Seguridad de BD** | Usuario `gpximport` con acceso directo a PostgreSQL | Acceso a través del ORM Rails |
| **Mantenimiento** | Dependencias C del sistema (libexpat, libgd, etc.) | Gema Ruby `gpx` |
| **Observabilidad** | Log file separado | Integrado en el stack Rails |


---
## 4. Arquitectura Actual

### 4.1 Stack tecnológico
El procesamiento de GPX ahora forma parte de **[`openstreetmap-website`](https://github.com/openstreetmap/openstreetmap-website)**, la aplicación Rails que alimenta todo openstreetmap.org. Los componentes clave son:
- **[`app/controllers/traces_controller.rb`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/traces_controller.rb)** — Maneja la subida y gestión de trazas
- **[`app/models/trace.rb`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/models/trace.rb)** — Lógica de negocio, parseo y almacenamiento
- **[`app/jobs/trace_importer_job.rb`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/jobs/trace_importer_job.rb)** — Job asíncrono de importación ([PR #2120](https://github.com/openstreetmap/openstreetmap-website/pull/2120))
- **[`app/jobs/trace_destroyer_job.rb`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/jobs/trace_destroyer_job.rb)** — Job asíncrono de eliminación
- **Gema [`gpx`](https://github.com/dougfales/gpx)** — Parsing del formato GPX en Ruby
- **ActiveJob** — Framework de colas de trabajo de Rails
- **AWS S3** — Almacenamiento de archivos GPX e imágenes
### 4.2 Modelo de datos (`Trace`)
La tabla subyacente sigue llamándose `gpx_files` (compatibilidad histórica):
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
#  inserted    :boolean          not null   # false = pendiente de importar
#  visibility  :enum             default("public")
#                                # valores: private | public | trackable | identifiable
```
**Relaciones:**
- `has_many :tags` (tabla `gpx_file_tags`)
- `has_many :points` (tabla `tracepoints`)
- `has_one_attached :file` → S3 bucket `openstreetmap-gps-traces`
- `has_one_attached :image` → S3 bucket `openstreetmap-gps-images`
- `has_one_attached :icon` → S3 bucket `openstreetmap-gps-images`
**Visibilidades disponibles:**
| Valor | Descripción |
|---|---|
| `private` | Solo visible para el dueño |
| `public` | Visible públicamente, sin identificación de usuario |
| `trackable` | Pública con timestamps, sin nombre de usuario |
| `identifiable` | Pública con timestamps y nombre de usuario |
### 4.3 Flujo técnico completo
#### Paso 1 — Subida del archivo ([`TracesController#create`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/controllers/traces_controller.rb))
```ruby
def create
  @trace = do_create(params[:trace][:gpx_file],
                     params[:trace][:tagstring],
                     params[:trace][:description],
                     params[:trace][:visibility])
  if @trace.id
    flash[:warning] = t(".traces_waiting", :count => ...) if pending_count > 4
    @trace.schedule_import   # ← encola el job
    redirect_to :action => :index
  end
end
```
**El método `do_create`:**
1. Sanitiza el nombre de archivo (reemplaza caracteres no alfanuméricos por `_`)
2. Detecta el tipo MIME real usando `/usr/bin/file` (no confía en la extensión)
3. Crea el registro `Trace` con `inserted: false` (marca como pendiente)
4. Sube el archivo a S3 mediante ActiveStorage
**Tipos de archivo soportados:**
| MIME type | Extensión lógica |
|---|---|
| `application/gpx+xml` | `.gpx` |
| `application/gzip` | `.gpx.gz` |
| `application/x-bzip2` | `.gpx.bz2` |
| `application/zip` | `.zip` |
| `application/x-tar+gzip` | `.tar.gz` |
| `application/x-tar+x-bzip2` | `.tar.bz2` |
| `application/x-tar` | `.tar` |
#### Paso 2 — Encolado del job ([`Trace#schedule_import`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/models/trace.rb))
```ruby
def schedule_import
  TraceImporterJob.new(self).enqueue(
    :priority => user.traces.where(:inserted => false).count
  )
end
```
La **prioridad es dinámica**: cuantas más trazas pendientes tenga el usuario, mayor es el número (menor prioridad), evitando que un usuario sature la cola.
#### Paso 3 — Ejecución del job ([`TraceImporterJob#perform`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/jobs/trace_importer_job.rb))
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
    UserMailer.gpx_failure(trace, "#{e}\\n#{e.backtrace.join("\\n")}").deliver
    trace.destroy
  end
end
```
El job maneja tres escenarios:
- **Éxito:** notifica al usuario con el conteo de puntos importados
- **Fallo de parseo XML:** notifica y elimina el trace
- **Error inesperado:** notifica con stack trace y elimina el trace
#### Paso 4 — Importación real ([`Trace#import`](https://github.com/openstreetmap/openstreetmap-website/blob/master/app/models/trace.rb))
```ruby
def import
  file.open do |file|
    gpx = GPX::File.new(file.path, :maximum_points => Settings.max_trace_size)
    # 1. Elimina puntos existentes (re-importación segura)
    Tracepoint.where(:trace => id).delete_all
    # 2. Importa puntos en lotes de 1.000
    gpx.points.each_slice(1_000) do |points|
      tracepoints = points.map do |point|
        Tracepoint.new(
          lat:       point.latitude,
          lon:       point.longitude,
          altitude:  point.altitude,
          timestamp: point.timestamp,
          gpx_id:    id,
          trackid:   point.segment   # ← preserva segmentos de traza
        )
      end
      Tracepoint.import!(tracepoints)  # bulk insert con activerecord-import (PR #2131)
    end
    # 3. Calcula bounding box y genera imágenes
    if gpx.actual_points.positive?
      # ... cálculo de min/max lat/lon desde la DB ...
      image.attach(:io => gpx.picture(min_lat, min_lon, max_lat, max_lon, actual_points),
                   :filename => "#{id}.gif", :content_type => "image/gif")
      icon.attach(:io => gpx.icon(min_lat, min_lon, max_lat, max_lon),
                  :filename => "#{id}_icon.gif", :content_type => "image/gif")
      self.size    = gpx.actual_points
      self.inserted = true   # ← marca como procesado
      save!
    end
  end
end
```
**Aspectos destacables:**
- Límite configurable de puntos por traza (`Settings.max_trace_size`)
- Inserción en batches de 1.000 puntos para eficiencia
- Preserva el número de segmento (`trackid`) de cada punto
- Genera dos imágenes: una de tamaño normal y un icono thumbnail (ambos GIF)
- Guarda la posición del **primer punto** (`latitude`, `longitude`) para geolocalización rápida
### 4.4 Infraestructura actual (Chef)
El worker de jobs Rails se configura como un **servicio systemd parametrizado** en [`cookbooks/web/recipes/rails.rb`](https://github.com/openstreetmap/chef/blob/master/cookbooks/web/recipes/rails.rb):
```ruby
systemd_service "rails-jobs@" do
  description "Rails job queue runner"
  type        "simple"
  environment "RAILS_ENV"   => "production",
              "QUEUE"       => "%I",       # ← parámetro: nombre de la cola
              "SLEEP_DELAY" => "60",
              "SECRET_KEY_BASE" => web_passwords["secret_key_base"]
  user              "rails"
  working_directory rails_directory
  exec_start        "#{node[:ruby][:bundle]} exec rails jobs:work"
  restart           "on-failure"
  nice              10        # menor prioridad de CPU
  sandbox           :enable_network => true
  memory_deny_write_execute false
  read_write_paths  "/var/log/web"
end
```
El servicio `rails-jobs@traces` procesa específicamente la cola de importación GPX. El parámetro `%I` en systemd se reemplaza por el nombre de la cola al instanciar el servicio, permitiendo múltiples workers para distintas colas.
**Almacenamiento en S3:**
| Bucket | Contenido | ACL |
|---|---|---|
| `openstreetmap-gps-traces` | Archivos GPX originales | `public-read` |
| `openstreetmap-gps-images` | Imágenes y thumbnails | `public-read` |
Región: `eu-west-1` (Irlanda), con `use_dualstack_endpoint: true` (IPv4 + IPv6).
---
## 5. Diagrama Comparativo de Arquitecturas
```
════════════════════════════════════════════════════════════════
  ARQUITECTURA ANTERIOR (hasta 2019)
════════════════════════════════════════════════════════════════
  [Usuario]
      │ HTTP upload
      ▼
  [Rails App]
      │ Guarda en /store/rails/gpx/traces (disco local)
      │ Marca trace como inserted=false en DB
      ▼
  [gpx-import daemon — proceso C separado]
      │ Polling cada 40 segundos
      │ Lee archivos del disco local
      │ Conexión directa a PostgreSQL (usuario gpximport)
      │ Genera imágenes → disco local
      ▼
  [PostgreSQL] ← acceso directo sin ORM
════════════════════════════════════════════════════════════════
  ARQUITECTURA ACTUAL (desde 2019)
════════════════════════════════════════════════════════════════
  [Usuario]
      │ HTTP upload
      ▼
  [Rails App — TracesController]
      │ Sube archivo a S3 (ActiveStorage)
      │ Crea registro Trace (inserted=false)
      │ Encola TraceImporterJob (prioridad dinámica)
      ▼
  [ActiveJob Worker — rails-jobs@traces]
      │ Descarga archivo de S3
      │ Parsea con gema Ruby 'gpx'
      │ Inserta Tracepoints en bulk (batches de 1.000)
      │ Genera imágenes GIF → sube a S3
      │ Actualiza Trace (inserted=true)
      ▼
  [PostgreSQL] ← a través del ORM Rails (ActiveRecord)
  [AWS S3]     ← archivos GPX + imágenes
```
---
## 6. Tabla Comparativa Final
| Aspecto | Arquitectura anterior | Arquitectura actual |
|---|---|---|
| **Implementación** | [Daemon C externo](https://github.com/openstreetmap/gpx-import) | ActiveJob en Rails ([PR #2120](https://github.com/openstreetmap/openstreetmap-website/pull/2120)) |
| **Parsing GPX** | [libexpat](https://github.com/openstreetmap/gpx-import/blob/master/src/main.c) (C) | Gema [`gpx`](https://github.com/dougfales/gpx) (Ruby) |
| **Activación** | Polling cada 40 s | Job encolado en el momento del upload |
| **Prioridad de cola** | No existía | Dinámica según trazas pendientes del usuario |
| **Almacenamiento de archivos** | Sistema de archivos local | AWS S3 (eu-west-1) — [config Chef](https://github.com/openstreetmap/chef/blob/master/cookbooks/web/recipes/rails.rb) |
| **Almacenamiento de imágenes** | Sistema de archivos local | AWS S3 (eu-west-1) — [config Chef](https://github.com/openstreetmap/chef/blob/master/cookbooks/web/recipes/rails.rb) |
| **Acceso a BD** | Conexión directa (usuario `gpximport`) | ORM ActiveRecord (usuario `rails`) |
| **Servidor dedicado** | Sí (rol [`web-gpximport`](https://github.com/openstreetmap/chef/blob/a7d96c8/roles/web-gpximport.rb)) | No (corre en servidores web existentes) |
| **Gestión de dependencias** | `apt` + compilación en producción | Bundler (Gemfile) |
| **Manejo de errores** | Log file + email | Excepciones Ruby + email + destrucción del trace |
| **Límite de puntos** | Hardcoded en C | Configurable (`Settings.max_trace_size`) |
| **Formatos soportados** | GPX, gz, bz2, zip, tar | GPX, gz, bz2, zip, tar.gz, tar.bz2 |
| **Inserción de puntos** | Fila por fila | Bulk insert en batches de 1.000 ([PR #2131](https://github.com/openstreetmap/openstreetmap-website/pull/2131)) |
| **Observabilidad** | Log file separado | Stack Rails integrado |
---
## 7. Relevancia para la Propuesta
Esta investigación demuestra que OpenStreetMap recorrió un camino de **simplificación arquitectónica progresiva**: eliminó un componente especializado en C que requería infraestructura dedicada, y lo integró al monolito Rails aprovechando sus mecanismos nativos (ActiveJob, ActiveStorage).
Los puntos de aprendizaje clave para considerar en la propuesta son:
1. **La integración es preferible a la especialización prematura.** Un daemon externo en C solo se justifica cuando Ruby no puede manejar la carga. En este caso, Ruby con bulk inserts resultó suficiente.
2. **ActiveJob desacopla el procesamiento sin añadir infraestructura.** El mismo worker que procesa otras colas de Rails puede procesar GPX, reduciendo la superficie operativa.
3. **S3 como capa de almacenamiento desacoplada** elimina dependencias de disco local y facilita el escalado horizontal de los servidores de aplicación.
4. **La prioridad dinámica de cola** es un mecanismo sencillo pero efectivo para prevenir abuso (un usuario que sube miles de archivos no bloquea a otros).
5. **El fork [`e-n-f/gpx-import`](https://github.com/e-n-f/gpx-import)** con su adición de vectores de dirección nunca fue integrado, lo que sugiere que esa funcionalidad no fue considerada prioritaria por la comunidad OSM en su momento.