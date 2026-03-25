# UML - MongoDB (Variante B) en Mermaid

## 1) Diagrama de flujo (alto nivel)
```mermaid
flowchart LR
  A[Kafka\n(datos_crudos, alertas_globales)] --> B[HDFS Raw\nlanding JSONL]
  B --> C[Spark 01\nraw_to_staging]
  C --> D[Hive\nlogistica.stg_ships]
  D --> E[Spark 04\nmongo_ship_latest\n(upsert ship_latest)]

  B --> F[Spark 03\nscore_and_alert]
  F --> G[Hive\nlogistica.fact_alerts]
  G --> H[Spark 05\nmongo_alerts_latest_by_route\n(upsert alerts_latest_by_route)]
```

## 2) Secuencia (micro-lote)
```mermaid
sequenceDiagram
  participant Airflow as Airflow
  participant Spark01 as Spark01
  participant Hive as Hive
  participant Mongo as Mongo
  participant Spark03 as Spark03
  participant Spark04 as Spark04
  participant Spark05 as Spark05

  Airflow->>Spark01: spark-submit 01_raw_to_staging
  Spark01->>Hive: escribe logistica.stg_ships
  Spark01-->>Airflow: OK

  Airflow->>Spark04: spark-submit 04_mongo_ship_latest
  Spark04->>Hive: lee logistica.stg_ships
  Spark04->>Mongo: upsert ship_latest (latest por ship_id)
  Spark04-->>Airflow: OK

  Airflow->>Spark03: spark-submit 03_score_and_alert
  Spark03->>Hive: escribe logistica.fact_alerts
  Spark03-->>Airflow: OK

  Airflow->>Spark05: spark-submit 05_mongo_alerts_latest_by_route
  Spark05->>Hive: lee logistica.fact_alerts
  Spark05->>Mongo: upsert alerts_latest_by_route (latest por via_port)
  Spark05-->>Airflow: OK
```

## 3) Diagrama de clases (modelo de datos Mongo)
```mermaid
classDiagram
  class ShipLatest {
    ship_id (key)
    ts
    route_id
    origin_port
    dest_port
    lat
    lon
    speed_kn
    heading
    warehouse
  }

  class AlertsLatestByRoute {
    via_port (key)
    alert_ts
    risk_score
    risk_level
    severity
    estimated_total_hours
    stock_on_hand_min
    reorder_point
    stock_status
    recommendation
  }

  class StgShip {
    ship_id
    ts
    route_id
    origin_port
    dest_port
    lat
    lon
    speed_kn
    heading
    warehouse
  }

  class FactAlert {
    via_port
    alert_ts
    risk_score
    risk_level
    severity
    estimated_total_hours
    stock_on_hand_min
    reorder_point
    stock_status
    recommendation
  }

  StgShip --> ShipLatest
  FactAlert --> AlertsLatestByRoute
```

## 4) Diagrama de casos de uso (Variante B)
```mermaid
usecaseDiagram
  actor Usuario as U
  actor Airflow as A
  actor Spark as S
  actor Mongo as M

  A --> "Ejecutar micro-lote\n01_raw_to_staging" as UC1
  S --> "Calcular latest\nship_id" as UC1a
  M --> "Upsert ship_latest" as UC1b

  A --> "Ejecutar micro-lote\n03_score_and_alert" as UC2
  S --> "Calcular latest\nvia_port" as UC2a
  M --> "Upsert alerts_latest_by_route" as UC2b

  U --> "Consultar ship_latest\npor ship_id" as UC3
  U --> "Consultar alerts_latest\npor via_port" as UC4
```

