# Requisitos - MongoDB (Variante B)

## 1. Objetivo
Persistir en MongoDB el **último estado** por vehículo (`ship_id`) y el **último estado de alertas** por puerto/ruta intermedia (`via_port`), sustituyendo el rol que tendría Cassandra para consultas de baja latencia.

## 2. Alcance
- Integración **después de que Spark haga limpieza/preprocesamiento**.
- Entradas principales:
  - `logistica.stg_ships` (generada por `jobs/spark/01_raw_to_staging.py`)
  - `logistica.fact_alerts` (generada por `jobs/spark/03_score_and_alert.py`)

## 3. Suposiciones
- `logistica.stg_ships` contiene un campo `ts` comparable para determinar el “latest” por `ship_id`.
- `logistica.fact_alerts` contiene:
  - `via_port`
  - `alert_ts` (timestamp de la alerta)
- MongoDB estará accesible desde el entorno donde se ejecuten los pasos que escriben en Mongo (idealmente `master`/Spark driver).

## 4. Requisitos funcionales
- FR1. `ship_latest` debe tener **1 documento por `ship_id`**.
- FR2. Para cada `ship_id`, `ship_latest` debe guardar el registro con **`ts` más reciente**.
- FR3. `alerts_latest_by_route` debe tener **1 documento por `via_port`**.
- FR4. Para cada `via_port`, `alerts_latest_by_route` debe guardar el registro con **`alert_ts` más reciente**.
- FR5. La actualización por micro-lote debe ser **idempotente**:
  - se implementa mediante `upsert` (por clave) y/o filtrado del “latest” antes de escribir.

## 5. Variantes (decisión)
Implementamos la variante **B**:
- Mongo:
  - `ship_latest` (latest por `ship_id`)
  - `alerts_latest_by_route` (latest por `via_port`)

## 6. Requisitos no funcionales
- NFR1. Observabilidad: log con número de documentos seleccionados/escritos y errores.
- NFR2. Resiliencia operacional: fallo de Mongo debe fallar el paso con error claro (o reintentar si Airflow lo configura).
- NFR3. Consistencia eventual: Mongo se actualiza al ritmo del DAG (micro-lotes), no en tiempo real.

## 7. Esquemas recomendados (Mongo)

### 7.1 Colección `ship_latest`
Campos mínimos:
- `ship_id` (clave de upsert)
- `ts`
- `route_id`
- `origin_port`, `dest_port`
- `lat`, `lon`
- `speed_kn`, `heading`
- `warehouse`

### 7.2 Colección `alerts_latest_by_route`
Campos mínimos:
- `via_port` (clave de upsert)
- `alert_ts`
- `risk_score`, `risk_level`
- `severity`
- `estimated_total_hours`
- `stock_on_hand_min`, `reorder_point`
- `stock_status`
- `recommendation`

## 8. Configuración
- `MONGO_URI`
- `MONGO_DB` (por ejemplo `logistica`)
- (opcional) `MONGO_COLLECTION_SHIPS_LATEST`, `MONGO_COLLECTION_ALERTS_LATEST_BY_ROUTE`

