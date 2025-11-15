# 資料模型設計

**日期**: 2025-11-15
**階段**: Phase 1 - Design & Contracts
**資料庫**: PostgreSQL 15+

## 概述

本文件定義 LINE Bot Zoom 控制功能所需的資料模型。系統使用 PostgreSQL 資料庫儲存會議記錄、錄製工作、使用者操作日誌與確認對話框狀態。

## 實體關係圖 (ERD)

```
┌─────────────────────┐
│  meeting_records    │
│                     │
│  - id (PK)          │
│  - meeting_id       │
│  - password         │
│  - display_name     │
│  - user_line_id (FK)│
│  - metadata (JSONB) │
│  - used_at          │
└─────────────────────┘
          │
          │ 1:N (一個使用者有多筆會議記錄)
          │
          ▼
┌─────────────────────┐         ┌─────────────────────┐
│ recording_sessions  │         │ confirmation_sessions│
│                     │         │                     │
│  - id (PK)          │         │  - id (PK)          │
│  - session_id (UK)  │         │  - session_id (UK)  │
│  - user_line_id (FK)│         │  - user_line_id (FK)│
│  - file_path        │         │  - action_type      │
│  - file_size        │         │  - action_payload   │
│  - duration_seconds │         │  - status           │
│  - status           │         │  - expires_at       │
│  - started_at       │         │  - created_at       │
│  - stopped_at       │         └─────────────────────┘
└─────────────────────┘
          │
          │ 1:N (一個使用者有多筆錄製記錄)
          │
          ▼
┌─────────────────────┐
│ user_action_logs    │
│                     │
│  - id (PK)          │
│  - user_line_id (FK)│
│  - action_type      │
│  - request_payload  │
│  - response_payload │
│  - status           │
│  - error_message    │
│  - created_at       │
└─────────────────────┘
```

**關聯說明**:
- `user_line_id` 作為外鍵（Foreign Key）串連所有實體
- 使用 LINE User ID 作為使用者識別（無需額外建立 users 表）

---

## 實體定義

### 1. meeting_records - 會議記錄

**用途**: 儲存使用者最近加入過的 Zoom 會議資訊，支援「快速重新加入」功能（FR-006, FR-007）。

**Schema**:

```sql
CREATE TABLE meeting_records (
    id                SERIAL PRIMARY KEY,
    meeting_id        VARCHAR(50) NOT NULL,
    password          VARCHAR(100),
    display_name      VARCHAR(100),
    user_line_id      VARCHAR(100) NOT NULL,
    metadata          JSONB,
    used_at           TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    -- 索引：查詢使用者的最近 5 筆會議記錄
    INDEX idx_user_used_at (user_line_id, used_at DESC)
);

-- 註解
COMMENT ON TABLE meeting_records IS '使用者加入過的 Zoom 會議記錄';
COMMENT ON COLUMN meeting_records.metadata IS '額外資訊（例如會議主題、參與人數等），JSONB 格式';
```

**欄位說明**:

| 欄位 | 型別 | 必填 | 說明 | 驗證規則 |
|------|------|------|------|----------|
| `id` | SERIAL | ✅ | 主鍵 | 自動遞增 |
| `meeting_id` | VARCHAR(50) | ✅ | Zoom 會議 ID | 純數字，長度 9-11 位 |
| `password` | VARCHAR(100) | ❌ | 會議密碼 | 可為空（無密碼會議） |
| `display_name` | VARCHAR(100) | ❌ | 顯示名稱 | 可為空（使用預設名稱） |
| `user_line_id` | VARCHAR(100) | ✅ | LINE 使用者 ID | 格式：`U` + 32 個字元 |
| `metadata` | JSONB | ❌ | 額外資訊 | JSON 格式，可擴展 |
| `used_at` | TIMESTAMP WITH TIME ZONE | ✅ | 使用時間 | 自動設為當前時間 |

**業務規則**:
- 每個使用者最多保留最近 5 筆記錄（應用層實作清理邏輯）
- 重複加入同一會議時，更新 `used_at` 欄位（或新增一筆記錄）
- 密碼欄位**不加密儲存**（因使用者可能重新加入同一會議，需明文密碼）

**安全性考量**:
- 密碼明文儲存風險：緩解措施為限制資料庫存取權限（僅 Gateway Service 可存取）
- 敏感資訊審計：所有讀取操作記錄至 `user_action_logs`

**範例資料**:
```sql
INSERT INTO meeting_records (meeting_id, password, display_name, user_line_id, metadata) VALUES
('1234567890', 'abc123', 'Allen', 'Uxxxxxxxxxxxxxxxxxxxxxxxxxxxx', '{"topic": "每週會議", "participants": 5}'),
('9876543210', NULL, 'Allen', 'Uxxxxxxxxxxxxxxxxxxxxxxxxxxxx', NULL);
```

---

### 2. recording_sessions - 錄製工作

**用途**: 追蹤螢幕錄製工作的生命週期，儲存錄製檔案資訊（FR-016, FR-017）。

**Schema**:

```sql
CREATE TABLE recording_sessions (
    id                SERIAL PRIMARY KEY,
    session_id        VARCHAR(100) UNIQUE NOT NULL,
    user_line_id      VARCHAR(100) NOT NULL,
    file_path         VARCHAR(500),
    file_size         BIGINT,
    duration_seconds  INT,
    status            VARCHAR(20) NOT NULL DEFAULT 'recording',
    started_at        TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    stopped_at        TIMESTAMP WITH TIME ZONE,

    -- 索引：查詢使用者的錄製歷史
    INDEX idx_user_started_at (user_line_id, started_at DESC),

    -- 約束：狀態必須為有效值
    CHECK (status IN ('recording', 'completed', 'failed'))
);

COMMENT ON TABLE recording_sessions IS '螢幕錄製工作記錄';
COMMENT ON COLUMN recording_sessions.session_id IS '唯一錄製 ID，格式：rec_YYYYMMDD_HHMMSS';
COMMENT ON COLUMN recording_sessions.status IS 'recording: 錄製中, completed: 已完成, failed: 失敗';
```

**欄位說明**:

| 欄位 | 型別 | 必填 | 說明 | 驗證規則 |
|------|------|------|------|----------|
| `id` | SERIAL | ✅ | 主鍵 | 自動遞增 |
| `session_id` | VARCHAR(100) | ✅ | 唯一錄製 ID | 格式：`rec_YYYYMMDD_HHMMSS`<br>範例：`rec_20251115_143022` |
| `user_line_id` | VARCHAR(100) | ✅ | LINE 使用者 ID | 格式：`U` + 32 個字元 |
| `file_path` | VARCHAR(500) | ❌ | 錄製檔案路徑 | 絕對路徑，例如：`/Users/mac/recordings/rec_xxx.mp4` |
| `file_size` | BIGINT | ❌ | 檔案大小（bytes） | 停止錄製後更新 |
| `duration_seconds` | INT | ❌ | 錄製時長（秒） | 停止錄製後計算 |
| `status` | VARCHAR(20) | ✅ | 錄製狀態 | `recording` \| `completed` \| `failed` |
| `started_at` | TIMESTAMP WITH TIME ZONE | ✅ | 開始時間 | 自動設為當前時間 |
| `stopped_at` | TIMESTAMP WITH TIME ZONE | ❌ | 停止時間 | 停止錄製時更新 |

**狀態轉換圖**:

```
┌───────────┐  開始錄製   ┌───────────┐
│  (不存在)  │ ────────→  │ recording │
└───────────┘             └─────┬─────┘
                                │
                      ┌─────────┼─────────┐
                      │                   │
                成功停止錄製          錄製失敗
                      │                   │
                      ▼                   ▼
                ┌───────────┐       ┌──────────┐
                │ completed │       │  failed  │
                └───────────┘       └──────────┘
                  (終止狀態)          (終止狀態)
```

**業務規則**:
- 同一時間僅允許一個錄製工作（應用層檢查：查詢是否有 `status = 'recording'` 的記錄）
- `session_id` 生成規則：`rec_` + 時間戳記（`20251115_143022`）
- 錄製失敗時，`error_message` 記錄至 `user_action_logs`

**範例資料**:
```sql
-- 正在錄製中
INSERT INTO recording_sessions (session_id, user_line_id, file_path, status) VALUES
('rec_20251115_143022', 'Uxxxxxxxxxxxxxxxxxxxxxxxxxxxx', '/Users/mac/recordings/rec_20251115_143022.mp4', 'recording');

-- 已完成的錄製
INSERT INTO recording_sessions (session_id, user_line_id, file_path, file_size, duration_seconds, status, started_at, stopped_at) VALUES
('rec_20251114_100000', 'Uxxxxxxxxxxxxxxxxxxxxxxxxxxxx', '/Users/mac/recordings/rec_20251114_100000.mp4', 2147483648, 932, 'completed', '2025-11-14 10:00:00+08', '2025-11-14 10:15:32+08');
```

---

### 3. user_action_logs - 使用者操作記錄

**用途**: 審計日誌，記錄所有使用者透過 LINE Bot 執行的操作（FR-028, FR-033）。

**Schema**:

```sql
CREATE TABLE user_action_logs (
    id                 SERIAL PRIMARY KEY,
    user_line_id       VARCHAR(100) NOT NULL,
    action_type        VARCHAR(50) NOT NULL,
    request_payload    JSONB,
    response_payload   JSONB,
    status             VARCHAR(20) NOT NULL,
    error_message      TEXT,
    created_at         TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    -- 索引：查詢使用者操作歷史
    INDEX idx_user_action_time (user_line_id, created_at DESC),

    -- 索引：查詢特定操作類型
    INDEX idx_action_type (action_type, created_at DESC),

    -- 約束：狀態必須為有效值
    CHECK (status IN ('success', 'error'))
);

COMMENT ON TABLE user_action_logs IS '使用者操作審計日誌';
COMMENT ON COLUMN user_action_logs.action_type IS '操作類型：zoom_join, zoom_leave, recording_start, recording_stop, status_query, screenshot';
COMMENT ON COLUMN user_action_logs.request_payload IS 'API 請求參數（JSONB 格式）';
COMMENT ON COLUMN user_action_logs.response_payload IS 'API 回應內容（JSONB 格式）';
```

**欄位說明**:

| 欄位 | 型別 | 必填 | 說明 | 驗證規則 |
|------|------|------|------|----------|
| `id` | SERIAL | ✅ | 主鍵 | 自動遞增 |
| `user_line_id` | VARCHAR(100) | ✅ | LINE 使用者 ID | 格式：`U` + 32 個字元 |
| `action_type` | VARCHAR(50) | ✅ | 操作類型 | 見下方「操作類型列表」 |
| `request_payload` | JSONB | ❌ | 請求參數 | JSON 格式，包含會議 ID、密碼等 |
| `response_payload` | JSONB | ❌ | 回應內容 | JSON 格式，包含執行結果 |
| `status` | VARCHAR(20) | ✅ | 執行狀態 | `success` \| `error` |
| `error_message` | TEXT | ❌ | 錯誤訊息 | 僅在 `status = 'error'` 時填寫 |
| `created_at` | TIMESTAMP WITH TIME ZONE | ✅ | 操作時間 | 自動設為當前時間 |

**操作類型列表**:

| action_type | 說明 | request_payload 範例 |
|-------------|------|---------------------|
| `zoom_join` | 加入 Zoom 會議 | `{"meeting_id": "1234567890", "password": "abc123"}` |
| `zoom_leave` | 離開 Zoom 會議 | `{}` |
| `recording_start` | 開始螢幕錄製 | `{}` |
| `recording_stop` | 停止螢幕錄製 | `{"session_id": "rec_20251115_143022"}` |
| `status_query` | 查詢系統狀態 | `{}` |
| `screenshot` | 取得螢幕截圖 | `{}` |

**業務規則**:
- 所有操作（成功或失敗）都必須記錄
- 日誌保留期限：30 天（應用層定期清理，或使用 PostgreSQL partitioning）
- 敏感資訊（密碼）在 `request_payload` 中記錄為 `***`（遮罩處理）

**安全性考量**:
- 密碼遮罩：`request_payload` 中的 `password` 欄位儲存為 `"password": "***"`
- 存取控制：僅 Gateway Service 有寫入權限，管理員有唯讀權限

**範例資料**:
```sql
-- 成功操作
INSERT INTO user_action_logs (user_line_id, action_type, request_payload, response_payload, status) VALUES
('Uxxxxxxxxxxxxxxxxxxxxxxxxxxxx', 'zoom_join', '{"meeting_id": "1234567890", "password": "***"}', '{"status": "success", "message": "已加入會議"}', 'success');

-- 失敗操作
INSERT INTO user_action_logs (user_line_id, action_type, request_payload, response_payload, status, error_message) VALUES
('Uxxxxxxxxxxxxxxxxxxxxxxxxxxxx', 'zoom_join', '{"meeting_id": "invalid", "password": "***"}', '{"status": "error"}', 'error', 'AppleScript execution failed: meeting ID invalid');
```

---

### 4. confirmation_sessions - 確認對話框狀態

**用途**: 管理使用者確認對話框的生命週期，支援 60 秒超時機制（FR-010, FR-011, FR-024, FR-025）。

**Schema**:

```sql
CREATE TABLE confirmation_sessions (
    id                SERIAL PRIMARY KEY,
    session_id        VARCHAR(100) UNIQUE NOT NULL,
    user_line_id      VARCHAR(100) NOT NULL,
    action_type       VARCHAR(50) NOT NULL,
    action_payload    JSONB NOT NULL,
    status            VARCHAR(20) DEFAULT 'pending',
    expires_at        TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at        TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    -- 索引：查詢使用者待確認的對話框
    INDEX idx_user_pending (user_line_id, status, expires_at),

    -- 約束：狀態必須為有效值
    CHECK (status IN ('pending', 'confirmed', 'cancelled', 'expired'))
);

COMMENT ON TABLE confirmation_sessions IS '確認對話框狀態管理';
COMMENT ON COLUMN confirmation_sessions.session_id IS '唯一確認 ID，格式：confirm_YYYYMMDD_HHMMSS';
COMMENT ON COLUMN confirmation_sessions.action_type IS '待確認操作：recording_start, recording_stop, zoom_leave';
COMMENT ON COLUMN confirmation_sessions.status IS 'pending: 待確認, confirmed: 已確認, cancelled: 已取消, expired: 已過期';
```

**欄位說明**:

| 欄位 | 型別 | 必填 | 說明 | 驗證規則 |
|------|------|------|------|----------|
| `id` | SERIAL | ✅ | 主鍵 | 自動遞增 |
| `session_id` | VARCHAR(100) | ✅ | 唯一確認 ID | 格式：`confirm_YYYYMMDD_HHMMSS` |
| `user_line_id` | VARCHAR(100) | ✅ | LINE 使用者 ID | 格式：`U` + 32 個字元 |
| `action_type` | VARCHAR(50) | ✅ | 待確認操作 | `recording_start` \| `recording_stop` \| `zoom_leave` |
| `action_payload` | JSONB | ✅ | 操作參數 | 執行操作所需的完整參數 |
| `status` | VARCHAR(20) | ✅ | 確認狀態 | `pending` \| `confirmed` \| `cancelled` \| `expired` |
| `expires_at` | TIMESTAMP WITH TIME ZONE | ✅ | 過期時間 | `created_at + 60 秒` |
| `created_at` | TIMESTAMP WITH TIME ZONE | ✅ | 建立時間 | 自動設為當前時間 |

**狀態轉換圖**:

```
┌───────────┐  建立確認對話框  ┌─────────┐
│  (不存在)  │ ─────────────→  │ pending │
└───────────┘                  └────┬────┘
                                    │
                      ┌─────────────┼─────────────┐
                      │             │             │
                  使用者確認     使用者取消      60秒超時
                      │             │             │
                      ▼             ▼             ▼
                ┌──────────┐  ┌──────────┐  ┌─────────┐
                │confirmed │  │cancelled │  │ expired │
                └──────────┘  └──────────┘  └─────────┘
                 (終止狀態)     (終止狀態)     (終止狀態)
```

**業務規則**:
- 確認對話框建立時，`expires_at = NOW() + INTERVAL '60 seconds'`
- 背景工作（cron job）定期掃描過期記錄，將 `status = 'pending' AND expires_at < NOW()` 的記錄更新為 `expired`
- 使用者確認後，應用層執行實際操作（加入會議、開始錄製等）

**範例資料**:
```sql
-- 待確認的對話框
INSERT INTO confirmation_sessions (session_id, user_line_id, action_type, action_payload, expires_at) VALUES
('confirm_20251115_143022', 'Uxxxxxxxxxxxxxxxxxxxxxxxxxxxx', 'recording_start', '{}', NOW() + INTERVAL '60 seconds');

-- 已確認的對話框
INSERT INTO confirmation_sessions (session_id, user_line_id, action_type, action_payload, status, expires_at) VALUES
('confirm_20251115_140000', 'Uxxxxxxxxxxxxxxxxxxxxxxxxxxxx', 'zoom_leave', '{}', 'confirmed', NOW() + INTERVAL '60 seconds');
```

---

## 索引策略

### 查詢效能優化

| 表格 | 索引 | 目的 | 預期查詢 |
|------|------|------|----------|
| `meeting_records` | `idx_user_used_at (user_line_id, used_at DESC)` | 查詢使用者最近 5 筆會議 | `SELECT * FROM meeting_records WHERE user_line_id = ? ORDER BY used_at DESC LIMIT 5` |
| `recording_sessions` | `idx_user_started_at (user_line_id, started_at DESC)` | 查詢使用者錄製歷史 | `SELECT * FROM recording_sessions WHERE user_line_id = ? ORDER BY started_at DESC` |
| `user_action_logs` | `idx_user_action_time (user_line_id, created_at DESC)` | 查詢使用者操作記錄 | `SELECT * FROM user_action_logs WHERE user_line_id = ? ORDER BY created_at DESC LIMIT 100` |
| `user_action_logs` | `idx_action_type (action_type, created_at DESC)` | 查詢特定操作類型的所有記錄（用於監控） | `SELECT * FROM user_action_logs WHERE action_type = 'zoom_join' AND created_at > ?` |
| `confirmation_sessions` | `idx_user_pending (user_line_id, status, expires_at)` | 查詢使用者待確認的對話框 | `SELECT * FROM confirmation_sessions WHERE user_line_id = ? AND status = 'pending'` |

---

## 資料保留政策

| 表格 | 保留期限 | 清理策略 |
|------|----------|----------|
| `meeting_records` | 每個使用者最多 5 筆 | 應用層實作：插入新記錄後刪除最舊記錄 |
| `recording_sessions` | 無限制（檔案手動清理） | 不自動刪除（錄製檔案需手動管理） |
| `user_action_logs` | 30 天 | PostgreSQL Partitioning + 定期刪除舊分區 |
| `confirmation_sessions` | 7 天 | Cron job 每日清理 `created_at < NOW() - INTERVAL '7 days'` |

**實作建議**:
```sql
-- 清理 30 天前的審計日誌（每日執行）
DELETE FROM user_action_logs WHERE created_at < NOW() - INTERVAL '30 days';

-- 清理 7 天前的確認對話框記錄（每日執行）
DELETE FROM confirmation_sessions WHERE created_at < NOW() - INTERVAL '7 days';

-- 清理使用者超過 5 筆的會議記錄（插入新記錄後執行）
DELETE FROM meeting_records
WHERE id IN (
    SELECT id FROM meeting_records
    WHERE user_line_id = ?
    ORDER BY used_at DESC
    OFFSET 5
);
```

---

## Migration 範例

**檔案**: `gateway/internal/db/migrations/001_initial_schema.sql`

```sql
-- 會議記錄表
CREATE TABLE meeting_records (
    id                SERIAL PRIMARY KEY,
    meeting_id        VARCHAR(50) NOT NULL,
    password          VARCHAR(100),
    display_name      VARCHAR(100),
    user_line_id      VARCHAR(100) NOT NULL,
    metadata          JSONB,
    used_at           TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
CREATE INDEX idx_user_used_at ON meeting_records(user_line_id, used_at DESC);

-- 錄製工作表
CREATE TABLE recording_sessions (
    id                SERIAL PRIMARY KEY,
    session_id        VARCHAR(100) UNIQUE NOT NULL,
    user_line_id      VARCHAR(100) NOT NULL,
    file_path         VARCHAR(500),
    file_size         BIGINT,
    duration_seconds  INT,
    status            VARCHAR(20) NOT NULL DEFAULT 'recording',
    started_at        TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    stopped_at        TIMESTAMP WITH TIME ZONE,
    CHECK (status IN ('recording', 'completed', 'failed'))
);
CREATE INDEX idx_user_started_at ON recording_sessions(user_line_id, started_at DESC);

-- 使用者操作日誌表
CREATE TABLE user_action_logs (
    id                 SERIAL PRIMARY KEY,
    user_line_id       VARCHAR(100) NOT NULL,
    action_type        VARCHAR(50) NOT NULL,
    request_payload    JSONB,
    response_payload   JSONB,
    status             VARCHAR(20) NOT NULL,
    error_message      TEXT,
    created_at         TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CHECK (status IN ('success', 'error'))
);
CREATE INDEX idx_user_action_time ON user_action_logs(user_line_id, created_at DESC);
CREATE INDEX idx_action_type ON user_action_logs(action_type, created_at DESC);

-- 確認對話框狀態表
CREATE TABLE confirmation_sessions (
    id                SERIAL PRIMARY KEY,
    session_id        VARCHAR(100) UNIQUE NOT NULL,
    user_line_id      VARCHAR(100) NOT NULL,
    action_type       VARCHAR(50) NOT NULL,
    action_payload    JSONB NOT NULL,
    status            VARCHAR(20) DEFAULT 'pending',
    expires_at        TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at        TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CHECK (status IN ('pending', 'confirmed', 'cancelled', 'expired'))
);
CREATE INDEX idx_user_pending ON confirmation_sessions(user_line_id, status, expires_at);
```

---

## Golang 模型定義

**檔案**: `gateway/internal/models/meeting.go`

```go
package models

import "time"

type MeetingRecord struct {
    ID          int64                  `xorm:"pk autoincr" json:"id"`
    MeetingID   string                 `xorm:"varchar(50) notnull" json:"meeting_id"`
    Password    string                 `xorm:"varchar(100)" json:"password,omitempty"`
    DisplayName string                 `xorm:"varchar(100)" json:"display_name"`
    UserLineID  string                 `xorm:"varchar(100) notnull" json:"user_line_id"`
    Metadata    map[string]interface{} `xorm:"jsonb" json:"metadata,omitempty"`
    UsedAt      time.Time              `xorm:"created" json:"used_at"`
}

func (MeetingRecord) TableName() string {
    return "meeting_records"
}
```

**檔案**: `gateway/internal/models/recording.go`

```go
package models

import "time"

type RecordingSession struct {
    ID              int64     `xorm:"pk autoincr" json:"id"`
    SessionID       string    `xorm:"varchar(100) unique notnull" json:"session_id"`
    UserLineID      string    `xorm:"varchar(100) notnull" json:"user_line_id"`
    FilePath        string    `xorm:"varchar(500)" json:"file_path,omitempty"`
    FileSize        int64     `xorm:"bigint" json:"file_size,omitempty"`
    DurationSeconds int       `xorm:"int" json:"duration_seconds,omitempty"`
    Status          string    `xorm:"varchar(20) notnull" json:"status"`
    StartedAt       time.Time `xorm:"created" json:"started_at"`
    StoppedAt       *time.Time `xorm:"timestamp" json:"stopped_at,omitempty"`
}

func (RecordingSession) TableName() string {
    return "recording_sessions"
}

const (
    RecordingStatusRecording = "recording"
    RecordingStatusCompleted = "completed"
    RecordingStatusFailed    = "failed"
)
```

---

## 總結

本資料模型設計符合以下原則：

✅ **正規化**: 避免資料重複，使用 JSONB 儲存彈性結構
✅ **效能優化**: 索引覆蓋常見查詢模式
✅ **安全性**: 敏感資訊（密碼）遮罩記錄至審計日誌
✅ **可擴展性**: 使用 JSONB metadata 欄位預留擴展空間
✅ **審計追蹤**: 完整記錄使用者操作與系統狀態變化

**下一步**: 定義 API 契約（OpenAPI 規格）。
