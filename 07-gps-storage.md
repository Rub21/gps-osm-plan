### Triple almacenamiento actual

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATOS GPS ACTUALES                            │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │     AWS S3       │  │   PostgreSQL     │  │  Tile Server  │  │
│  │                  │  │                  │  │  (muirdris)   │  │
│  │ Archivos GPX     │  │ Tabla            │  │               │  │
│  │ originales       │  │ tracepoints      │  │ GPX gzipped   │  │
│  │ (ActiveStorage)  │  │ (fila por fila)  │  │ + índice      │  │
│  │                  │  │                  │  │ datamaps      │  │
│  │ Propósito:       │  │ Propósito:       │  │               │  │
│  │ descarga usuario │  │ API trackpoints  │  │ Propósito:    │  │
│  │                  │  │                  │  │ tiles PNG     │  │
│  └──────────────────┘  └──────────────────┘  └───────────────┘  │
│                                                                  │
│  ⚠️  La tabla tracepoints es el cuello de botella de backups     │
└─────────────────────────────────────────────────────────────────┘
```