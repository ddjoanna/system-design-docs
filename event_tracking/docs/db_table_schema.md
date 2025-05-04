## 🧱 PostgreSQL Schema 設計

1. `tenants`：租戶定義 (如 企業、品牌、個人)
2. `applications`：註冊的應用程式（例如 Web App、Mobile App）
3. `platforms`：平台定義（如 Web、iOS、Android）
4. `events`：定義哪些事件要被追蹤
5. `event_fields`：定義事件所需欄位（欄位名稱 + 資料型態）
6. `sessions`：使用者 Session（匿名或具名）
7. `event_logs`：事件實際紀錄（不含欄位值，寫入 Kafka 後送至 OLAP 儲存）

---

```sql
-- 租戶資訊
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL UNIQUE, -- 租戶名稱（如公司名稱）
    description TEXT, -- 租戶描述
    created_at TIMESTAMP NOT NULL DEFAULT now(), -- 建立時間
    updated_at TIMESTAMP NOT NULL DEFAULT now(), -- 更新時間
    CONSTRAINT tenant_name_unique UNIQUE (name) -- 確保每個租戶的名稱唯一
);

COMMENT ON COLUMN tenants.name IS '租戶名稱（如公司名稱）';
COMMENT ON COLUMN tenants.description IS '租戶描述';
COMMENT ON COLUMN tenants.created_at IS '租戶創建的時間戳';
COMMENT ON COLUMN tenants.updated_at IS '租戶最後一次更新的時間戳';

CREATE UNIQUE INDEX idx_tenants_name ON tenants (name);


-- 應用程式註冊資訊
CREATE TABLE applications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE, -- 租戶 ID
    name TEXT NOT NULL, -- 應用程式名稱
    api_key TEXT UNIQUE NOT NULL, -- 應用程式的 API 密鑰
    description TEXT, -- 應用程式描述
    created_at TIMESTAMP NOT NULL DEFAULT now(), -- 應用程式創建時間
    updated_at TIMESTAMP NOT NULL DEFAULT now(), -- 應用程式更新時間
    CONSTRAINT tenant_app_name_unique UNIQUE (tenant_id, name) -- 確保每個租戶中的應用程式名稱唯一
);

COMMENT ON COLUMN applications.tenant_id IS '關聯的租戶 ID';
COMMENT ON COLUMN applications.name IS '應用程式名稱';
COMMENT ON COLUMN applications.api_key IS '應用程式的 API 密鑰';
COMMENT ON COLUMN applications.description IS '應用程式描述';
COMMENT ON COLUMN applications.created_at IS '應用程式創建時間';
COMMENT ON COLUMN applications.updated_at IS '應用程式更新時間';

CREATE UNIQUE INDEX idx_applications_tenant_id_name ON applications (tenant_id, name);
CREATE UNIQUE INDEX idx_applications_api_key ON applications (api_key);

-- 支援的平台種類
CREATE TABLE platforms (
    id SERIAL PRIMARY KEY, -- 平台 ID
    name TEXT NOT NULL UNIQUE, -- 平台名稱
    created_at TIMESTAMP NOT NULL DEFAULT now() -- 創建時間
);

COMMENT ON COLUMN platforms.name IS '平台名稱（如 Web, iOS, Android）';
COMMENT ON COLUMN platforms.created_at IS '平台創建時間';

CREATE UNIQUE INDEX idx_platforms_name ON platforms (name);


-- 定義事件類型
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES applications(id) ON DELETE CASCADE, -- 關聯的應用程式 ID
    platform_id INTEGER REFERENCES platforms(id), -- 關聯的平台 ID
    name TEXT NOT NULL, -- 事件名稱
    description TEXT, -- 事件描述
    is_active BOOLEAN DEFAULT TRUE, -- 事件是否啟用
    created_at TIMESTAMP NOT NULL DEFAULT now(), -- 事件創建時間
    updated_at TIMESTAMP NOT NULL DEFAULT now(), -- 事件更新時間
    CONSTRAINT app_event_name_unique UNIQUE (application_id, name) -- 確保每個應用程式內的事件名稱唯一
);

COMMENT ON COLUMN events.application_id IS '關聯的應用程式 ID';
COMMENT ON COLUMN events.platform_id IS '關聯的平台 ID';
COMMENT ON COLUMN events.name IS '事件名稱';
COMMENT ON COLUMN events.description IS '事件描述';
COMMENT ON COLUMN events.is_active IS '事件是否啟用';
COMMENT ON COLUMN events.created_at IS '事件創建時間';
COMMENT ON COLUMN events.updated_at IS '事件更新時間';

CREATE UNIQUE INDEX idx_events_application_id_name ON events (application_id, name);
CREATE INDEX idx_events_platform_id ON events (platform_id);

-- 每個事件定義其欄位與資料型別（僅 schema 定義）
CREATE TABLE event_fields (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE, -- 關聯的事件 ID
    name TEXT NOT NULL, -- 欄位名稱
    data_type TEXT NOT NULL CHECK (data_type IN ('string', 'int', 'float', 'boolean', 'datetime', 'json')), -- 資料型別
    is_required BOOLEAN DEFAULT FALSE, -- 欄位是否為必填
    description TEXT, -- 欄位描述
    created_at TIMESTAMP NOT NULL DEFAULT now(), -- 欄位創建時間
    CONSTRAINT event_field_unique UNIQUE (event_id, name) -- 確保每個事件中的欄位名稱唯一
);

COMMENT ON COLUMN event_fields.event_id IS '關聯的事件 ID';
COMMENT ON COLUMN event_fields.name IS '欄位名稱';
COMMENT ON COLUMN event_fields.data_type IS '資料型別';
COMMENT ON COLUMN event_fields.is_required IS '是否為必填欄位';
COMMENT ON COLUMN event_fields.description IS '欄位描述';
COMMENT ON COLUMN event_fields.created_at IS '欄位創建時間';

CREATE UNIQUE INDEX idx_event_fields_event_id_name ON event_fields (event_id, name);

-- Session，可由 SDK 初始化時建立，與匿名訪客或 user_id 綁定
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES applications(id) ON DELETE CASCADE, -- 關聯的應用程式 ID
    platform_id INTEGER NOT NULL REFERENCES platforms(id), -- 關聯的平台 ID
    session_key TEXT NOT NULL UNIQUE, -- Session 唯一鍵
    user_id TEXT, -- 使用者 ID（可選）
    user_agent TEXT, -- 使用者代理（瀏覽器 / 設備型號）
    ip_address TEXT, -- 使用者 IP 地址
    started_at TIMESTAMP NOT NULL DEFAULT now(), -- Session 開始時間
    ended_at TIMESTAMP, -- Session 結束時間
    created_at TIMESTAMP NOT NULL DEFAULT now() -- Session 創建時間
);

COMMENT ON COLUMN sessions.application_id IS '關聯的應用程式 ID';
COMMENT ON COLUMN sessions.platform_id IS '關聯的平台 ID';
COMMENT ON COLUMN sessions.session_key IS 'Session 唯一鍵';
COMMENT ON COLUMN sessions.user_id IS '使用者 ID（匿名或登入帳號 ID）';
COMMENT ON COLUMN sessions.user_agent IS '使用者代理';
COMMENT ON COLUMN sessions.ip_address IS '使用者 IP 地址';
COMMENT ON COLUMN sessions.started_at IS 'Session 開始時間';
COMMENT ON COLUMN sessions.ended_at IS 'Session 結束時間';
COMMENT ON COLUMN sessions.created_at IS 'Session 創建時間';

CREATE UNIQUE INDEX idx_sessions_session_key ON sessions (session_key);
CREATE INDEX idx_sessions_application_id_user_id ON sessions (application_id, user_id);

-- 實際事件記錄（僅紀錄主幹，欄位資料寫入 Kafka → ClickHouse）
CREATE TABLE event_logs (
    id UUID, -- 記錄唯一ID
    application_id UUID NOT NULL REFERENCES applications(id) ON DELETE CASCADE, -- 關聯的應用程式 ID
    session_id UUID REFERENCES sessions(id) ON DELETE SET NULL, -- 關聯的 Session ID
    event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE, -- 關聯的事件 ID
    platform_id INTEGER NOT NULL REFERENCES platforms(id), -- 關聯的平台 ID
    created_at TIMESTAMP NOT NULL DEFAULT now(), -- 記錄創建時間
    properties JSONB, -- 備援記錄（補充欄位資料）
    indexed BOOLEAN DEFAULT FALSE, -- 是否已經建立索引

    CONSTRAINT event_logs_pk PRIMARY KEY (id, created_at), -- 主鍵需要包含分區鍵
    CONSTRAINT event_logs_created_at CHECK (created_at <= now()) -- 防止未來日期插入
)
PARTITION BY RANGE (created_at); -- 按照 created_at 欄位的範圍進行分區

COMMENT ON COLUMN event_logs.application_id IS '關聯的應用程式 ID';
COMMENT ON COLUMN event_logs.session_id IS '關聯的 Session ID';
COMMENT ON COLUMN event_logs.event_id IS '關聯的事件 ID';
COMMENT ON COLUMN event_logs.platform_id IS '關聯的平台 ID';
COMMENT ON COLUMN event_logs.created_at IS '記錄創建時間';
COMMENT ON COLUMN event_logs.properties IS '事件的備援記錄';
COMMENT ON COLUMN event_logs.indexed IS '是否已經建立索引';

CREATE INDEX idx_event_logs_application_id_created_at ON event_logs (application_id, created_at);
CREATE INDEX idx_event_logs_session_id ON event_logs (session_id);
CREATE INDEX idx_event_logs_event_id ON event_logs (event_id);
CREATE INDEX idx_event_logs_platform_id ON event_logs (platform_id);

```

---

### 🔖 補充說明

- `event_logs.properties` 是備援記錄，主系統會將事件 JSON 送往 Kafka → ClickHouse。
- 可考慮建立部分索引欄位於 PostgreSQL，供簡單查詢或 Debug 使用。
- ClickHouse 將作為主要儲存與分析的資料倉庫。

---

## 🧱 ClickHouse Schema 設計

### 設計目標

- 儲存大量追蹤事件資料
- 支援自定義欄位（Flexible Schema）
- 可依時間、平台、事件類型快速查詢
- 供 OLAP 工具（如 Grafana、Metabase）整合

---

#### 1. **`tenants`**：租戶主表

```sql
CREATE TABLE tenants (
    id UUID DEFAULT generateUUIDv4(),
    name String,
    description String,
    created_at DateTime DEFAULT now(),
    updated_at DateTime DEFAULT now()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (id)
SETTINGS index_granularity = 8192;

-- Index 建議
CREATE INDEX idx_tenants_name ON tenants (name) TYPE minmax GRANULARITY 4;
```

#### 2. **`applications`**：每個租戶的應用程式

```sql
CREATE TABLE applications (
    id UUID DEFAULT generateUUIDv4(),
    tenant_id UUID,
    name String,
    api_key String,
    description String,
    created_at DateTime DEFAULT now(),
    updated_at DateTime DEFAULT now()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)  -- 依照創建時間分區
ORDER BY (tenant_id, id)           -- 依租戶ID排序，保證每個租戶的應用程式能夠獨立查詢
SETTINGS index_granularity = 8192;

-- Index 建議
CREATE INDEX idx_applications_tenant_id_name ON applications (tenant_id, name) TYPE minmax GRANULARITY 4;
CREATE INDEX idx_applications_api_key ON applications (api_key) TYPE minmax GRANULARITY 4;
```

#### 3. **`platforms`**：平台類型（Web, iOS, Android 等）

```sql
CREATE TABLE platforms (
    id UInt32,
    tenant_id UUID,  -- 使平台與租戶相關聯
    name String,
    created_at DateTime DEFAULT now()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (tenant_id, id)           -- 依租戶ID排序
SETTINGS index_granularity = 8192;

-- Index 建議
CREATE INDEX idx_platforms_tenant_id_name ON platforms (tenant_id, name) TYPE minmax GRANULARITY 4;
```

#### 4. **`events`**：事件定義

```sql
CREATE TABLE events (
    id UUID DEFAULT generateUUIDv4(),
    tenant_id UUID,
    application_id UUID,
    platform_id UInt32,
    name String,
    description String,
    is_active UInt8 DEFAULT 1,
    created_at DateTime DEFAULT now(),
    updated_at DateTime DEFAULT now()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (tenant_id, application_id, id)  -- 確保每個租戶的事件資料能夠高效查詢
SETTINGS index_granularity = 8192;

-- Index 建議
CREATE INDEX idx_events_tenant_id_application_id_name ON events (tenant_id, application_id, name) TYPE minmax GRANULARITY 4;
CREATE INDEX idx_events_platform_id ON events (platform_id) TYPE minmax GRANULARITY 4;
```

#### 5. **`event_fields`**：事件欄位定義

```sql
CREATE TABLE event_fields (
    id UUID DEFAULT generateUUIDv4(),
    tenant_id UUID,
    event_id UUID,
    name String,
    data_type String,
    is_required UInt8 DEFAULT 0,
    description String,
    created_at DateTime DEFAULT now()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (tenant_id, event_id, id)  -- 確保每個租戶的欄位資料能夠高效查詢
SETTINGS index_granularity = 8192;

-- Index 建議
CREATE INDEX idx_event_fields_tenant_id_event_id_name ON event_fields (tenant_id, event_id, name) TYPE minmax GRANULARITY 4;
```

#### 6. **`sessions`**：Session 定義

```sql
CREATE TABLE sessions (
    id UUID DEFAULT generateUUIDv4(),
    tenant_id UUID,
    application_id UUID,
    platform_id UInt32,
    session_key String,
    user_id String,
    user_agent String,
    ip_address String,
    started_at DateTime DEFAULT now(),
    ended_at DateTime,
    created_at DateTime DEFAULT now()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (tenant_id, application_id, id)  -- 確保每個租戶的會話資料能夠高效查詢
SETTINGS index_granularity = 8192;

-- Index 建議
CREATE INDEX idx_sessions_tenant_id_user_id ON sessions (tenant_id, user_id) TYPE minmax GRANULARITY 4;
CREATE INDEX idx_sessions_session_key ON sessions (session_key) TYPE minmax GRANULARITY 4;
```

#### 7. **`event_logs`**：事件日誌

```sql
CREATE TABLE event_logs (
    id UUID DEFAULT generateUUIDv4(),
    tenant_id UUID,
    application_id UUID,
    session_id UUID,
    event_id UUID,
    platform_id UInt32,
    created_at DateTime DEFAULT now(),
    properties JSONB,
    indexed UInt8 DEFAULT 0
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (tenant_id, application_id, event_id, created_at)  -- 確保每個租戶的事件日誌能夠高效查詢
SETTINGS index_granularity = 8192;

-- Index 建議
CREATE INDEX idx_event_logs_tenant_id_application_id_created_at ON event_logs (tenant_id, application_id, created_at) TYPE minmax GRANULARITY 4;
CREATE INDEX idx_event_logs_session_id ON event_logs (session_id) TYPE minmax GRANULARITY 4;
CREATE INDEX idx_event_logs_event_id ON event_logs (event_id) TYPE minmax GRANULARITY 4;
CREATE INDEX idx_event_logs_platform_id ON event_logs (platform_id) TYPE minmax GRANULARITY 4;
```

---

### **設計說明與建議**

1. **Partitioning**:

   - 所有表格都使用 `toYYYYMM(created_at)` 來進行 Partition，使得每個月的資料分為一個 Partition。這樣可以有效地將資料分區，便於高效查詢和數據管理。
   - 可以根據實際情況調整 Partition 策略，例如可以根據 `tenant_id` 來分區（尤其當有多租戶且查詢會集中於某些租戶時）。

2. **Indexing**:

   - **Minmax 索引**：使用 `minmax` 類型的索引來加速查詢範圍條件（如時間範圍、ID 範圍等）的查詢。
   - 每個表的主要查詢條件（如租戶、應用程式名稱、事件名稱、Session ID 等）都應該有索引以加速查詢。
   - 使用 `index_granularity` 設定來優化查詢性能，默認值為 8192，這是較為通用的配置。

3. **JSONB**：

   - `event_logs` 表中的 `properties` 欄位設為 `JSONB` 類型，允許靈活儲存事件屬性數據，並且便於處理不同事件的多變結構。

---

## 📦 📡 Kafka Topic 結構設計

透過 Apache NiFi Kafka → ClickHouse

### 1. Topic 命名規則

```
event_logs.{env}.{platform}
```

- `env`: prod, staging, dev
- `platform`: web, ios, android

例如：

- `event_logs.prod.web`
- `event_logs.dev.ios`

### 2. Kafka Message Format（JSON）

```json
{
  "tenant_id": "tenant_123", // 新增租戶ID
  "event_id": "uuid", // 事件ID
  "event_name": "product_click", // 事件名稱
  "application_id": "uuid", // 應用ID
  "platform": "web", // 平台類型
  "session_id": "uuid", // 會話ID
  "user_id": "anonymous", // 用戶ID
  "created_at": "2025-05-03T12:00:00Z", // 事件發生時間
  "ip_address": "192.168.1.1", // 用戶IP
  "user_agent": "Mozilla/5.0", // 用戶代理
  "properties": {
    "product_id": "P123", // 產品ID
    "category": "books", // 產品分類
    "price": 299 // 產品價格
  }
}
```

3. NiFi 可透過 `ReplaceText` → `ConvertRecord` → `PutClickHouseRecord` 寫入此資料表。

---

## 🔍 OLAP 分析建議查詢（範例）

在使用 ClickHouse 進行 OLAP (聯機分析處理) 分析時，建議設計查詢以支援高效的多維分析。以下是針對您的 schema 設計的常見 OLAP 查詢建議：

### 1. **按平台和事件類型統計事件次數**

此查詢可以幫助您統計不同平台上的事件類型數量，並提供基於平台的事件分析。

```sql
SELECT
    p.name AS platform_name,
    e.name AS event_name,
    COUNT(el.id) AS event_count
FROM event_logs el
JOIN platforms p ON el.platform_id = p.id
JOIN events e ON el.event_id = e.id
WHERE el.created_at >= '2025-01-01' AND el.created_at < '2025-02-01'
GROUP BY platform_name, event_name
ORDER BY event_count DESC
LIMIT 10;
```

### 2. **按租戶統計各平台事件次數**

此查詢有助於分析各租戶在不同平台上觸發的事件數量，適用於多租戶系統。

```sql
SELECT
    t.name AS tenant_name,
    p.name AS platform_name,
    COUNT(el.id) AS event_count
FROM event_logs el
JOIN applications a ON el.application_id = a.id
JOIN tenants t ON a.tenant_id = t.id
JOIN platforms p ON el.platform_id = p.id
WHERE el.created_at >= '2025-01-01' AND el.created_at < '2025-02-01'
GROUP BY tenant_name, platform_name
ORDER BY event_count DESC
LIMIT 10;
```

### 3. **按日期範圍分析事件發生頻率**

此查詢可以幫助您了解不同日期範圍內的事件頻率變化，適用於監控或趨勢分析。

```sql
SELECT
    toStartOfDay(el.created_at) AS day,
    COUNT(el.id) AS event_count
FROM event_logs el
WHERE el.created_at >= '2025-01-01' AND el.created_at < '2025-01-31'
GROUP BY day
ORDER BY day;
```

### 4. **根據 Session 統計每個用戶的事件觸發數量**

此查詢有助於分析每個用戶在其 session 中觸發的事件數量。

```sql
SELECT
    s.user_id,
    COUNT(el.id) AS event_count
FROM event_logs el
JOIN sessions s ON el.session_id = s.id
WHERE el.created_at >= '2025-01-01' AND el.created_at < '2025-02-01'
GROUP BY s.user_id
ORDER BY event_count DESC
LIMIT 10;
```

### 5. **按時間段統計事件數量和平台分佈**

此查詢提供時間段內事件的統計數據，並顯示每個平台上事件的分佈情況。

```sql
SELECT
    toStartOfHour(el.created_at) AS hour,
    p.name AS platform_name,
    COUNT(el.id) AS event_count
FROM event_logs el
JOIN platforms p ON el.platform_id = p.id
WHERE el.created_at >= '2025-01-01' AND el.created_at < '2025-01-10'
GROUP BY hour, platform_name
ORDER BY hour, event_count DESC;
```

### 6. **統計每個應用程式的事件總數**

這有助於了解每個應用程式的事件活動情況。

```sql
SELECT
    a.name AS application_name,
    COUNT(el.id) AS event_count
FROM event_logs el
JOIN applications a ON el.application_id = a.id
WHERE el.created_at >= '2025-01-01' AND el.created_at < '2025-02-01'
GROUP BY application_name
ORDER BY event_count DESC
LIMIT 10;
```

### 7. **根據事件欄位分析事件屬性**

如果您需要查看某些特定事件欄位的分佈情況，可以根據事件欄位進行篩選和統計。

```sql
SELECT
    event_field.name AS field_name,
    COUNT(el.id) AS event_count
FROM event_logs el
ARRAY JOIN el.properties AS event_field
WHERE el.created_at >= '2025-01-01' AND el.created_at < '2025-02-01'
GROUP BY field_name
ORDER BY event_count DESC
LIMIT 10;
```

### 8. **跨平台事件比較**

如果您希望比較兩個或更多平台上的事件觸發次數，可以使用 `JOIN` 查詢來實現。

```sql
SELECT
    p1.name AS platform_1,
    p2.name AS platform_2,
    COUNT(el1.id) AS platform_1_event_count,
    COUNT(el2.id) AS platform_2_event_count
FROM event_logs el1
JOIN platforms p1 ON el1.platform_id = p1.id
JOIN event_logs el2 ON el1.session_id = el2.session_id
JOIN platforms p2 ON el2.platform_id = p2.id
WHERE el1.created_at >= '2025-01-01' AND el1.created_at < '2025-02-01'
AND el2.created_at >= '2025-01-01' AND el2.created_at < '2025-02-01'
GROUP BY platform_1, platform_2
ORDER BY platform_1_event_count DESC, platform_2_event_count DESC;
```

### 9. **按 Session 計算每個 Session 的事件數量**

此查詢可以幫助您分析每個 session 中的事件觸發次數，適合用於分析用戶行為。

```sql
SELECT
    s.session_key,
    COUNT(el.id) AS event_count
FROM event_logs el
JOIN sessions s ON el.session_id = s.id
WHERE el.created_at >= '2025-01-01' AND el.created_at < '2025-02-01'
GROUP BY s.session_key
ORDER BY event_count DESC
LIMIT 10;
```

### 10. **全平台事件趨勢圖**

如果需要繪製多平台的事件趨勢，可以將不同平台的事件數量對比視覺化。

```sql
SELECT
    toStartOfDay(el.created_at) AS day,
    p.name AS platform_name,
    COUNT(el.id) AS event_count
FROM event_logs el
JOIN platforms p ON el.platform_id = p.id
WHERE el.created_at >= '2025-01-01' AND el.created_at < '2025-01-10'
GROUP BY day, platform_name
ORDER BY day, platform_name;
```

### OLAP 查詢設計要點：

1. **合理的 `PARTITION BY` 設定**：確保大數據量的表格能夠高效分割，通常按時間或其他維度分區。
2. **`JOIN` 設計**：基於多維度的查詢需求，合理設計 `JOIN` 操作，避免不必要的資料掃描。
3. **索引**：利用 `minmax` 和 `set` 索引來加速查詢，尤其是按時間範圍、平台等常用篩選條件進行查詢時。
4. **適當的數據彙總**：對常見的查詢進行聚合或預先計算，以減少查詢延遲。

這些查詢可作為您開始進行 OLAP 分析的基礎，並可根據需求進一步調整和擴展。

---

你的 Apache NiFi Dataflow 設計看起來非常完整，但我會做一些細微調整和修正，以確保流程順利運行，並在必要時澄清一些細節。這是調整後的版本：

---

## 🔄 **Apache NiFi Dataflow 設計**

### 🧩 **元件流程**

```
ConsumeKafka → ReplaceText → ConvertRecord → PutClickHouseRecord
```

### **詳細步驟說明：**

#### 1. `ConsumeKafka_2_0`

- **Topic Name**：`event_logs.*.*` (訂閱多個平台的事件資料)
- **Record Reader**：`JsonTreeReader`（解析 JSON 資料）
- **Record Writer**：`JsonRecordSetWriter`（寫入 JSON 格式）
- 自動訂閱所有平台的事件資料

#### 2. `ReplaceText`

- 可加上邏輯轉換 `created_at` 格式或補上缺失欄位（如 `tenant_id` 等）
- 用作臨時處理點，例如補充缺失的欄位或修正格式

#### 3. `ConvertRecord`

- **Record Reader**：`JsonTreeReader`
- **Record Writer**：`ClickHouseRecordSetWriter`
- 定義 ClickHouse 表結構對應的 schema，並轉換資料格式為 ClickHouse 兼容的格式

#### 4. `PutClickHouseRecord`

- **Database Connection Pooling Service**：設定 ClickHouse JDBC 連線池
- **Table Name**：`event_logs`
- **Batch Size**：500\~1000（根據流量調整）
- 可進一步調整寫入資料量以適應實際需求

---

## 🔐 **安全與高可用建議**

- **Kafka 配置**：

  - 設定 `message.max.bytes` 以支援較大的事件資料
  - 開啟 Kafka 的資料壓縮功能：推薦使用 `lz4` 或 `snappy`

- **NiFi 配置**：

  - NiFi 進行批次處理（不逐筆寫入），以提高效率並減少 ClickHouse 資料庫的負擔

- **ClickHouse 高可用設置**：

  - 啟用 Zookeeper 與 `ReplicatedMergeTree` 引擎來支援 ClickHouse 的高可用與容錯能力

---

## 📊 **Grafana 整合 ClickHouse**

1. 安裝 [ClickHouse Grafana 插件](https://grafana.com/grafana/plugins/vertamedia-clickhouse-datasource/)
2. 配置 ClickHouse Data Source
3. 範例查詢：

```sql
SELECT
  toStartOfHour(created_at) AS hour,
  count() AS total_events
FROM event_logs
WHERE event_name = 'page_view'
  AND event_date >= today() - 1
GROUP BY hour
ORDER BY hour
```

這個查詢可以幫助你查看最近一天內每小時的事件數量。

---

## 📝 **NiFi Flow Template XML 範本**

以下是 NiFi Flow Template 的範本，將會自動化 Kafka 消費、資料轉換及寫入 ClickHouse 的過程：

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<template>
  <name>Multi-Tenant Event Tracking Flow</name>
  <description>Consume Kafka → ConvertRecord → PutClickHouseRecord (with multi-tenant support)</description>
  <snippet>
    <connections/>
    <processors>
      <!-- Kafka Consumer -->
      <processor>
        <name>ConsumeKafka_2_0</name>
        <class>org.apache.nifi.kafka.pubsub.ConsumeKafka_2_0</class>
        <config>
          <properties>
            <property>
              <name>Kafka Topic Name</name>
              <value>event_logs.*.*</value>
            </property>
            <property>
              <name>Record Reader</name>
              <value>JsonTreeReader</value>
            </property>
            <property>
              <name>Record Writer</name>
              <value>JsonRecordSetWriter</value>
            </property>
            <property>
              <name>Group ID</name>
              <value>event-tracker-group</value>
            </property>
            <property>
              <name>Offset Reset</name>
              <value>earliest</value>
            </property>
          </properties>
        </config>
      </processor>

      <!-- Record Transformation (Convert to multi-tenant structure) -->
      <processor>
        <name>ConvertRecord</name>
        <class>org.apache.nifi.processors.standard.ConvertRecord</class>
        <config>
          <properties>
            <property>
              <name>Record Reader</name>
              <value>JsonTreeReader</value>
            </property>
            <property>
              <name>Record Writer</name>
              <value>ClickHouseRecordSetWriter</value>
            </property>
            <property>
              <name>Additional Field Mapping</name>
              <value>tenant_id = ${tenant_id}</value> <!-- Mapping tenant_id -->
            </property>
          </properties>
        </config>
      </processor>

      <!-- ClickHouse Sink -->
      <processor>
        <name>PutClickHouseRecord</name>
        <class>org.apache.nifi.clickhouse.PutClickHouseRecord</class>
        <config>
          <properties>
            <property>
              <name>Database Connection Pooling Service</name>
              <value>ClickHouseConnectionPool</value>
            </property>
            <property>
              <name>Table Name</name>
              <value>event_logs</value>
            </property>
            <property>
              <name>Batch Size</name>
              <value>500</value>
            </property>
            <property>
              <name>Additional Fields</name>
              <value>tenant_id, application_id, session_id, event_id, platform_id, created_at, properties</value> <!-- Ensure multi-tenant fields are inserted -->
            </property>
          </properties>
        </config>
      </processor>
    </processors>
  </snippet>
</template>
```

---

### **JSON Schema（供 JsonTreeReader 使用）**

```json
{
  "type": "record",
  "name": "EventLog",
  "fields": [
    { "name": "tenant_id", "type": "string" },
    { "name": "platform", "type": "string" },
    { "name": "session_id", "type": "string" },
    { "name": "event_key", "type": "string" },
    { "name": "event_version", "type": "string" },
    { "name": "event_time", "type": "string" },
    { "name": "attributes", "type": { "type": "map", "values": "string" } },
    { "name": "created_at", "type": "string" }
  ]
}
```

這份 JSON Schema 用來設置 `JsonTreeReader`，確保 NiFi 正確解析事件資料的各個欄位，並根據需要進行處理。
