# 行為追蹤事件服務 (Event Tracking Service)

本專案為一套典型的**事件追蹤分析系統**，架構類似 Segment、Mixpanel，使用 Golang + PostgreSQL 作為後端服務，Kafka 作為事件訊息佇列，Apache NiFi 負責資料處理，ClickHouse 作為 OLAP 分析資料庫，前端 React 透過自製 SDK 以 HTTP 傳送事件。

---

## 系統架構

```
Frontend (React) --> SDK (HTTP) --> Golang Tracking API --> Kafka --> NiFi --> ClickHouse --> Grafana
                              \
                               --> PostgreSQL (事件結構定義與管理)
```

---

## 功能說明

- **前端 SDK**  
  提供簡單易用的事件追蹤接口，將事件資料以 JSON 格式送至後端 API。

- **Golang Tracking API**  
  接收事件請求，驗證事件類型與欄位，非同步推送事件至 Kafka。

- **PostgreSQL**  
  管理事件類型與欄位定義，支援動態擴充與驗證。

- **Kafka**  
  高效能事件訊息佇列，確保事件資料可靠傳遞。

- **Apache NiFi**  
  消費 Kafka 事件，進行資料清洗、轉換，寫入 ClickHouse。

- **ClickHouse**  
  高速 OLAP 資料庫，支援大規模事件資料分析。

- **Grafana**  
  可視化事件分析報表。

---

## Table Schema (PostgreSQL)

### event_types

| 欄位名稱    | 類型      | 說明                     |
| ----------- | --------- | ------------------------ |
| id          | UUID      | 主鍵                     |
| name        | TEXT      | 事件名稱（註冊、點擊等） |
| description | TEXT      | 描述                     |
| created_at  | TIMESTAMP | 建立時間                 |

### event_fields

| 欄位名稱      | 類型      | 說明                          |
| ------------- | --------- | ----------------------------- |
| id            | UUID      | 主鍵                          |
| event_type_id | UUID      | 對應 event_types.id           |
| name          | TEXT      | 欄位名稱（如 button_name）    |
| data_type     | TEXT      | 欄位型別（string, int, bool） |
| required      | BOOL      | 是否必填                      |
| created_at    | TIMESTAMP | 建立時間                      |

---

## Kafka 訊息格式範例

```
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

## 前端 SDK 範例 (JavaScript)

```
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

## 技術選型

| 模組         | 技術/工具          | 說明                          |
| ------------ | ------------------ | ----------------------------- |
| 前端         | React + JS SDK     | 事件收集與送出                |
| API Server   | Golang (Gin/Fiber) | 事件接收、驗證、推送 Kafka    |
| 事件定義管理 | PostgreSQL         | 事件與欄位結構管理            |
| 快取         | Redis (可選)       | 快取事件定義，提高效能        |
| 訊息佇列     | Kafka              | 高併發事件資料緩衝與分發      |
| 資料處理     | Apache NiFi        | 事件資料清洗、轉換、寫入 OLAP |
| OLAP 資料庫  | ClickHouse         | 高速事件資料分析              |
| 可視化報表   | Grafana            | 事件分析儀表板                |

---

## 快速開始

1. 建立 PostgreSQL 資料庫並執行 schema SQL。
2. 啟動 Kafka 與 NiFi，設定資料流。
3. 部署 Golang API 服務，設定連接 Kafka 與 PostgreSQL。
4. 前端引入 SDK，開始發送事件。
5. 配置 ClickHouse 表結構，連接 Grafana 建立報表。

---

## 未來擴展建議

- 支援多種事件來源（Mobile SDK、Server SDK）。
- 事件資料加密與安全傳輸。
- 實時流式分析（如 Apache Flink）。
- 多租戶支援與權限控管。
- 事件版本管理與回溯。

---

## 聯絡方式

如有問題或建議，歡迎提出 Issue
