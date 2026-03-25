# Casos de uso - MongoDB (Variante B)

## 1. Actores
- Airflow (orquestador): ejecuta micro-lotes del DAG
- Spark (procesador): genera tablas Hive y calcula “latest”
- Hive (almacenamiento SQL): proporciona `stg_ships` y `fact_alerts`
- MongoDB (almacenamiento NoSQL): mantiene `ship_latest` y `alerts_latest_by_route`

## 2. Casos de uso

### UC-01. Actualizar `ship_latest`
- Descripción: cada micro-lote, Spark lee `logistica.stg_ships` y actualiza `ship_latest` en Mongo con el último registro por `ship_id`.
- Disparador: final de `jobs/spark/01_raw_to_staging.py`

### UC-02. Consultar último estado por `ship_id`
- Descripción: un consumidor (app/reporting) consulta Mongo para obtener el último estado de un barco.
- Entrada: `ship_id`
- Salida: documento desde `ship_latest`

### UC-03. Actualizar `alerts_latest_by_route`
- Descripción: cada micro-lote, Spark lee `logistica.fact_alerts` y actualiza `alerts_latest_by_route` en Mongo con el último registro por `via_port`.
- Disparador: final de `jobs/spark/03_score_and_alert.py`

### UC-04. Consultar última alerta por `via_port`
- Descripción: un consumidor (app/reporting) consulta Mongo para obtener la última alerta para un puerto/ruta intermedia.
- Entrada: `via_port`
- Salida: documento desde `alerts_latest_by_route`

## 3. Postcondiciones esperadas
- `ship_latest` contiene el último documento por cada `ship_id` observado.
- `alerts_latest_by_route` contiene el último documento por cada `via_port` observado.
- La actualización es idempotente por micro-lote (por `upsert`).

