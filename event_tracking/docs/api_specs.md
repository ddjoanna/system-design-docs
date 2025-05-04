## 📂 API 分類總覽

> 第一階段單純實現事件追蹤分析系統基本功能，暫時不實現 API Key 管理、帳號管理、權限管理

| 類別                | 功能項目              | 方法 | 路徑                                                                                 |
| ------------------- | --------------------- | ---- | ------------------------------------------------------------------------------------ |
| 🛠️ 管理 API         | 建立 App              | POST | /admin/apps                                                                          |
|                     | 查詢所有 App          | GET  | /admin/apps                                                                          |
|                     | 查詢 App 詳細         | GET  | /admin/apps/{app_key}                                                                |
|                     | 建立平台              | POST | /admin/apps/{app_key}/platforms                                                      |
|                     | 查詢平台              | GET  | /admin/apps/{app_key}/platforms                                                      |
|                     | 建立事件定義          | POST | /admin/platforms/{platform_code}/events                                              |
|                     | 查詢事件清單          | GET  | /sdk/{app_key}/{platform_code}/events                                                |
|                     | 新增事件欄位          | POST | /admin/events/{event_id}/fields                                                      |
|                     | 查詢事件欄位          | GET  | /admin/events/{event_id}/fields                                                      |
| 📦 SDK API          | 初始化 Session        | POST | /sdk/init-session                                                                    |
|                     | 傳送事件資料          | POST | /sdk/events                                                                          |
| 🧾 Session 查詢 API | 查詢 Session 清單     | GET  | /admin/tenants/{tenant_id}/sessions                                                  |
|                     | 查詢單一 Session 詳細 | GET  | /admin/tenants/{tenant_id}/sessions/{session_id}                                     |
| 📊 統計報表 API     | 活躍使用者統計        | GET  | /admin/tenants/{tenant_id}/analytics/active-users?start={start_date}\&end={end_date} |
|                     | 事件計數統計          | GET  | /admin/tenants/{tenant_id}/analytics/event-count?event_name={event_name}             |

---

## 📘 API Schema 總覽（多租戶事件追蹤系統）

### 🛠️ 管理 API（Admin）

#### 🔸 建立 App

```http
POST /admin/apps
```

**Request JSON:**

```json
{
  "app_key": "shop_app",
  "name": "My Shop",
  "description": "Shopping platform"
}
```

**Response JSON:**

```json
{
  "app_key": "shop_app",
  "name": "My Shop",
  "description": "Shopping platform",
  "created_at": "2025-05-04T10:00:00Z"
}
```

#### 🔸 查詢所有 App

```http
GET /admin/apps
```

**Response JSON:**

```json
[
  {
    "app_key": "shop_app",
    "name": "My Shop",
    "description": "Shopping platform"
  }
]
```

#### 🔸 查詢 App 詳細

```http
GET /admin/apps/{app_key}
```

**Response JSON:**

```json
{
  "app_key": "shop_app",
  "name": "My Shop",
  "description": "Shopping platform"
}
```

---

#### 🔸 建立平台

```http
POST /admin/apps/{app_key}/platforms
```

**Request JSON:**

```json
{
  "platform_code": "web",
  "name": "Web Platform"
}
```

**Response JSON:**

```json
{
  "platform_code": "web",
  "name": "Web Platform"
}
```

#### 🔸 查詢平台

```http
GET /admin/apps/{app_key}/platforms
```

**Response JSON:**

```json
[
  {
    "platform_code": "web",
    "name": "Web Platform"
  }
]
```

---

#### 🔸 建立事件定義

```http
POST /admin/platforms/{platform_code}/events
```

**Request JSON:**

```json
{
  "event_name": "click_button",
  "description": "Click on CTA"
}
```

**Response JSON:**

```json
{
  "event_id": "evt_001",
  "event_name": "click_button",
  "description": "Click on CTA"
}
```

#### 🔸 查詢事件清單

```http
GET /sdk/{app_key}/{platform_code}/events
```

**Response JSON:**

```json
[
  {
    "event_id": "evt_001",
    "event_name": "click_button",
    "description": "Click on CTA"
  }
]
```

---

#### 🔸 新增事件欄位

```http
POST /admin/events/{event_id}/fields
```

**Request JSON:**

```json
{
  "field_name": "button_id",
  "type": "string",
  "required": true
}
```

**Response JSON:**

```json
{
  "field_name": "button_id",
  "type": "string",
  "required": true
}
```

#### 🔸 查詢事件欄位

```http
GET /admin/events/{event_id}/fields
```

**Response JSON:**

```json
[
  {
    "field_name": "button_id",
    "type": "string",
    "required": true
  }
]
```

---

### 📦 SDK API

#### 🔸 初始化 Session

```http
POST /sdk/init-session
```

**Request JSON:**

```json
{
  "app_key": "shop_app",
  "platform_code": "web",
  "user_id": "u_123"
}
```

**Response JSON:**

```json
{
  "session_id": "s_abc123",
  "started_at": "2025-05-04T08:00:00Z"
}
```

#### 🔸 傳送事件資料

```http
POST /sdk/events
```

**Request JSON:**

```json
{
  "session_id": "s_abc123",
  "event_name": "click_button",
  "timestamp": "2025-05-04T08:30:00Z",
  "fields": {
    "button_id": "submit_btn"
  }
}
```

**Response JSON:**

```json
{
  "status": "success"
}
```

---

### 🧾 Session 查詢 API

#### 🔸 查詢 Session 清單

```http
GET /admin/tenants/{tenant_id}/sessions
```

**Response JSON:**

```json
[
  {
    "session_id": "s_abc123",
    "user_id": "u_12345",
    "started_at": "2025-05-04T08:00:00Z"
  }
]
```

#### 🔸 查詢單一 Session 詳細資料

```http
GET /admin/tenants/{tenant_id}/sessions/{session_id}
```

**Response JSON:**

```json
{
  "session_id": "s_abc123",
  "user_id": "u_12345",
  "events": [
    {
      "event_name": "click_banner",
      "timestamp": "2025-05-04T08:01:00Z",
      "fields": {
        "banner_id": "b_001"
      }
    }
  ]
}
```

---

### 📊 統計報表 API

#### 🔸 活躍使用者統計

```http
GET /admin/tenants/{tenant_id}/analytics/active-users?start=2025-05-01&end=2025-05-04
```

**Response JSON:**

```json
{
  "active_users": 2310
}
```

#### 🔸 事件計數統計

```http
GET /admin/tenants/{tenant_id}/analytics/event-count?event_name=click_button
```

**Response JSON:**

```json
{
  "event_name": "click_button",
  "count": 12845
}
```
