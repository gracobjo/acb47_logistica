# Diseño - MongoDB (Variante B)

## 1. Flujo lógico (integración)
La variante B añade **pasos de escritura en Mongo** en puntos concretos del pipeline:

1. Tras `jobs/spark/01_raw_to_staging.py`
   - Hive: `logistica.stg_ships`
   - Se construye `ship_latest` (latest por `ship_id`) en Mongo

2. Tras `jobs/spark/03_score_and_alert.py`
   - Hive: `logistica.fact_alerts`
   - Se construye `alerts_latest_by_route` (latest por `via_port`) en Mongo

## 2. Implementación recomendada
Crear nuevos jobs Spark siguiendo el estilo actual del repo:

- `jobs/spark/04_mongo_ship_latest.py`
- `jobs/spark/05_mongo_alerts_latest_by_route.py`

Estos jobs:
- crean `SparkSession` con soporte Hive,
- leen las tablas Hive,
- calculan “latest” por clave con ventana (`window` / `row_number`),
- aplican `upsert` en Mongo.

## 3. Selección determinista de “latest”

### 3.1 `ship_latest` (latest por `ship_id`)
- Particionar por `ship_id`
- Ordenar por `ts` descendente
- Quedarse con `row_number() == 1`

Nota: si `ts` en `stg_ships` es texto ISO, convertir a timestamp antes de ordenar para robustez.

### 3.2 `alerts_latest_by_route` (latest por `via_port`)
- Particionar por `via_port`
- Ordenar por `alert_ts` descendente
- Quedarse con `row_number() == 1`

## 4. Claves de upsert (Mongo)
- `ship_latest`
  - clave: `ship_id`
  - operación: `update_one({ship_id: ...}, {$set: doc}, upsert=true)`

- `alerts_latest_by_route`
  - clave: `via_port`
  - operación: `update_one({via_port: ...}, {$set: doc}, upsert=true)`

## 5. Integración en Airflow
Modificar `airflow/dags/logistica_kdd_dag.py` para añadir tareas:
- `spark_mongo_ship_latest` ejecutando `spark-submit jobs/spark/04_mongo_ship_latest.py`
- `spark_mongo_alerts_latest_by_route` ejecutando `spark-submit jobs/spark/05_mongo_alerts_latest_by_route.py`

Encadenamiento recomendado:
- `01_raw_to_staging` -> `04_mongo_ship_latest`
- `02_graph_metrics` -> `03_score_and_alert` -> `05_mongo_alerts_latest_by_route`

## 6. Manejo de errores y observabilidad
- Registrar:
  - nº de filas “latest” calculadas
  - nº de upserts exitosos
  - nº de errores (y ejemplos)
- Si Mongo falla: que el job falle con excepción para que Airflow registre el problema.

## 7. Dependencias
- Añadir dependencia a librería de Mongo en el entorno donde corran los Spark jobs (por ejemplo `pymongo`).

