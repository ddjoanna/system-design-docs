## Context Diagram（系統上下文圖）

```mermaid
graph TB
    subgraph User
        FE[React Frontend App]
    end

    subgraph System[Event Tracking System]
        SDK[JavaScript SDK]
        API[Golang Tracking API]
        PG[PostgreSQL: Event Schema]
        KAFKA[Kafka]
        NIFI[Apache NiFi]
        CH[ClickHouse]
        GRAFANA[Grafana Dashboard]
    end

    FE --> SDK
    SDK -->|HTTP POST /track| API
    API --> PG
    API --> KAFKA
    KAFKA --> NIFI
    NIFI --> CH
    CH --> GRAFANA
```

---

## Container Diagram（系統容器圖）

```mermaid
graph TD
    subgraph Frontend
        FE[React App]
        SDK[Tracking SDK JS]
    end

    subgraph Backend
        API[Golang REST API<br/>- Validate Events<br/>- Push to Kafka]
        AdminAPI[Admin API<br/>- Manage event_types & fields]
        PG[PostgreSQL<br/>event_types / event_fields]
        Redis[Redis Cache<br/>- Cached event schema]
    end

    subgraph DataPipeline
        Kafka[Kafka<br/>Topic: tracking_events]
        NiFi[Apache NiFi<br/>- Validate / Transform / Enrich]
        CH[ClickHouse<br/>- Raw events storage]
    end

    subgraph Analytics
        Grafana[Grafana<br/>OLAP Dashboard]
    end

    FE --> SDK
    SDK -->|POST /track| API
    API --> Redis
    API --> PG
    API --> Kafka
    AdminAPI --> PG
    Kafka --> NiFi
    NiFi --> CH
    CH --> Grafana
```

---

## Component Diagram 系統組件圖

### Tracking API 組件

```mermaid
graph TB
    subgraph Tracking API Golang
        Ctrl[TrackController<br/>接收 HTTP 請求]
        Validator[EventValidator<br/>檢查欄位是否符合 schema]
        Service[EventService<br/>組合封包與邏輯處理]
        Producer[KafkaProducer<br/>送出事件訊息]
        Repo[EventSchemaRepo<br/>從 Redis/Postgres 取得欄位定義]
    end

    Ctrl --> Validator
    Ctrl --> Service
    Validator --> Repo
    Service --> Producer
    Repo --> PG[PostgreSQL]
    Repo --> Redis
```

### Tracking API 架構

```mermaid
graph TD
    subgraph Web/API Layer
        HTTPHandler[HTTP Handler\ntrack_handler.go, admin_handler.go]
    end

    subgraph Application Layer UseCase
        TrackEventUC[TrackEventUseCase]
        AdminSchemaUC[ManageEventSchemaUseCase]
    end

    subgraph Domain Layer
        EventModel[Event Entity]
        SchemaModel[Schema Entity]
        SchemaValidator[Schema Validator Service]
    end

    subgraph Infrastructure Layer
        PGRepo[PostgreSQL Repository\nevent_repo.go]
        RedisRepo[Redis Cache\nschema_cache.go]
        KafkaProducer[Kafka Publisher\nkafka_producer.go]
    end

    subgraph External Systems
        PostgreSQL[PostgreSQL]
        Redis[Redis]
        Kafka[Kafka]
    end

    %% Connections
    HTTPHandler --> TrackEventUC
    HTTPHandler --> AdminSchemaUC

    TrackEventUC --> EventModel
    TrackEventUC --> SchemaValidator
    TrackEventUC --> RedisRepo
    TrackEventUC --> PGRepo
    TrackEventUC --> KafkaProducer

    AdminSchemaUC --> SchemaModel
    AdminSchemaUC --> PGRepo
    AdminSchemaUC --> RedisRepo

    PGRepo --> PostgreSQL
    RedisRepo --> Redis
    KafkaProducer --> Kafka

    SchemaValidator --> SchemaModel
```

#### 📌 每個組件功能說明

| 組件                         | 描述                                                          |
| ---------------------------- | ------------------------------------------------------------- |
| **HTTP Handler**             | 提供 HTTP API 入口，驗證參數後呼叫 UseCase                    |
| **TrackEventUseCase**        | 處理前端送來的事件資料，套用 schema 驗證後發送到 Kafka 並紀錄 |
| **ManageEventSchemaUseCase** | 提供 UI 後台新增、修改、刪除事件 Schema 的功能                |
| **Event Entity**             | 描述使用者送出的追蹤事件物件                                  |
| **Schema Entity**            | 管理每種事件對應的欄位定義與驗證規則                          |
| **Schema Validator**         | 核對事件資料與對應 Schema 欄位的型別與約束                    |
| **PostgreSQL Repository**    | 儲存事件資料、schema 設定，提供 CRUD 功能                     |
| **Redis Cache**              | 快取 Schema 加速查詢（避免頻繁查資料庫）                      |
| **Kafka Producer**           | 將追蹤事件封裝後發送到 Kafka Topic，供下游處理                |

---

## Code Diagram 系統程式碼圖

```mermaid
graph TD
    subgraph Interfaces
        HTTP[HTTP Handler controller]
    end

    subgraph UseCase
        TrackUC[TrackEventUseCase]
        AdminUC[ManageEventSchemaUseCase]
    end

    subgraph Domain
        EventEntity[Event Entity]
        SchemaEntity[Schema Entity]
    end

    subgraph Infrastructure
        PGRepo[Postgres Repository]
        RedisRepo[Redis Repository]
        KafkaPub[Kafka Producer]
    end

    HTTP --> TrackUC
    HTTP --> AdminUC
    TrackUC --> EventEntity
    TrackUC --> RedisRepo
    TrackUC --> PGRepo
    TrackUC --> KafkaPub

    AdminUC --> SchemaEntity
    AdminUC --> PGRepo

    EventEntity -->|Validate fields| SchemaEntity
```
