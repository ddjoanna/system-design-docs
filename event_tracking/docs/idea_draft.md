# 事件追蹤分析系統初步想法整理

包含系統架構圖、服務設計說明、Table Schema 及技術選型建議：

---

## 系統架構圖（高層次）

```
+--------------------+      +-----------------------+
|   Frontend (React) | ---> |   SDK (HTTP via JS)   |
+--------------------+      +-----------+-----------+
                                         |
                                         v
                        +-------------------------------+
                        |     Golang Tracking API       |
                        |  - 接收事件 / 驗證 / 整理資料   |
                        |  - 傳送到 Kafka (async)       |
                        +-------------------------------+
                                         |
                                         v
                            +--------------------+
                            |      Kafka         |
                            |  - Event Queue     |
                            +--------------------+
                                         |
                                         v
                                +----------------+
                                |     NiFi       |
                                | - 處理 / 清洗    |
                                | - 輸出到 OLAP    |
                                +--------+--------+
                                         |
                                         v
                             +------------------------+
                             |      ClickHouse        |
                             |   - 儲存事件資料        |
                             +------------------------+
                                         |
                                         v
                               +-------------------+
                               |     Grafana       |
                               |  - OLAP 可視化     |
                               +-------------------+

        +---------------------------+
        | PostgreSQL (控制用)       |
        | - event_type 設定         |
        | - event_fields 設定       |
        +---------------------------+
```

---

## 各服務設計說明

### Tracking API（Golang）

- 提供 `POST /track` 接口接收前端事件。
- 驗證事件類型 (`event_type`) 是否存在，欄位是否符合 PostgreSQL 中 `event_fields` 定義。
- 可用 Redis 快取事件定義提升效能。
- 事件資料經格式化後非同步推送至 Kafka。

### PostgreSQL（事件結構定義）

- 儲存事件類型與欄位設定，供 API 驗證與管理。
- 提供管理介面或 API 讓 Admin 新增、修改事件與欄位。

### Kafka（訊息佇列）

- 高併發事件資料的緩衝與分發，確保事件不遺失。
- 建議使用單一 Topic（如 `tracking_events`），由 NiFi 消費。

### Apache NiFi（資料轉換）

- 從 Kafka 消費事件資料。
- 執行資料清洗、欄位補齊（如 IP 解析）、格式轉換。
- 將處理後資料寫入 ClickHouse。

### ClickHouse（OLAP 儲存）

- 儲存結構化事件資料，支援高速聚合與分析查詢。
- 建議使用 Wide Schema，並以 `event_date` 分區提升查詢效率。

### Grafana（報表分析）

- 連接 ClickHouse，建立基於事件類型的可視化儀表板。
- 支援即時與歷史行為分析。

---

## Table Schema 設計（PostgreSQL）

### `event_types`

| 欄位名稱    | 類型      | 說明                     |
| ----------- | --------- | ------------------------ |
| id          | UUID      | 主鍵                     |
| name        | TEXT      | 事件名稱（註冊、點擊等） |
| description | TEXT      | 描述                     |
| created_at  | TIMESTAMP | 建立時間                 |

```sql
CREATE TABLE event_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL UNIQUE,
    description TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

### `event_fields`

| 欄位名稱      | 類型      | 說明                          |
| ------------- | --------- | ----------------------------- |
| id            | UUID      | 主鍵                          |
| event_type_id | UUID      | 對應 event_types.id           |
| name          | TEXT      | 欄位名稱（如 button_name）    |
| data_type     | TEXT      | 欄位型別（string, int, bool） |
| required      | BOOL      | 是否必填                      |
| created_at    | TIMESTAMP | 建立時間                      |

```sql
CREATE TABLE event_fields (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type_id UUID NOT NULL REFERENCES event_types(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    data_type TEXT NOT NULL,
    required BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

---

## Kafka 訊息格式建議

```json
{
  "event_type": "button_click",
  "timestamp": "2025-05-03T10:00:00Z",
  "user_id": "abc-123",
  "session_id": "xyz-456",
  "fields": {
    "button_name": "submit",
    "page_url": "/checkout"
  },
  "ip": "192.168.1.1",
  "user_agent": "Mozilla/5.0 ..."
}
```

---

## 前端 SDK 設計（JavaScript）

```js
window.trackEvent = async function (eventType, fields) {
  await fetch("https://track.yourdomain.com/track", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      event_type: eventType,
      fields,
      timestamp: new Date().toISOString(),
      session_id: getSessionId(),
    }),
  });
};
```

---

## 技術選型建議與補充

- **Golang**：高效能 API 實作，支援高併發。
- **PostgreSQL**：管理事件與欄位元資料，關聯資料庫適合結構化管理。
- **Redis**（可選）：快取事件定義，減少 DB 查詢延遲。
- **Kafka**：事件訊息佇列，確保資料可靠與可擴展。
- **Apache NiFi**：靈活資料流處理與轉換，易於擴展與管理。
- **ClickHouse**：高效 OLAP 分析，適合大規模事件資料查詢。
- **Grafana**：強大且靈活的可視化分析工具。
- **事件驗證**：可用 Golang 搭配 JSON Schema 進行欄位驗證。
- **ClickHouse 表設計**：建議以 `event_date` 作為分區鍵，加速查詢。
- **Kafka Topic**：單一 Topic 管理多事件類型，簡化架構。
