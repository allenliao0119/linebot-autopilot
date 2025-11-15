# 技術決策記錄：Zoom 會議與螢幕錄製控制

**日期**: 2025-11-14
**狀態**: 待進入 Planning 階段時整合到 plan.md
**決策者**: 開發團隊

> **注意**: 此文件記錄技術討論和決策理由，將在執行 `/speckit.plan` 時整合到正式的實作計畫中。

---

## 核心問題

**挑戰**: 本地 Mac 電腦位於家用網路環境，無固定 IP，且不適合直接暴露到公網。LINE Bot 需要固定 IP 的 webhook endpoint。

**問題**:
1. 如何讓 LINE Bot 安全地控制位於私有網路的 Mac？
2. 如何避免複雜的路由器設定（port forwarding）？
3. 如何確保通訊安全且穩定？

---

## 系統架構決策

### ✅ 決策：雲端 Gateway + 本地 Agent 架構（透過 Tailscale）

**架構概述**:
```
LINE Platform → AWS Gateway (固定 IP) → Tailscale VPN → 本地 Mac Service
```

**分層設計**:

1. **LINE Bot 層**:
   - LINE Messaging API
   - Webhook 推送到雲端固定 IP

2. **雲端 Gateway 層** (AWS):
   - 接收並驗證 LINE webhook
   - 處理使用者授權和確認對話框狀態
   - 轉發指令到本地 Mac
   - 回傳執行結果給 LINE 使用者
   - 審計日誌記錄

3. **Tailscale VPN 層**:
   - 建立雲端與本地的加密隧道
   - 自動 NAT 穿透
   - 提供虛擬 IP (100.x.x.x)

4. **本地 Mac Control Service 層**:
   - 暴露 REST API 供 Gateway 呼叫
   - 控制 Zoom 應用程式（AppleScript）
   - 控制螢幕錄製（ffmpeg + BlackHole）
   - 提供狀態查詢和螢幕截圖

**架構圖**:
```
┌──────────────────────────────────────────────────────────────┐
│                        LINE Platform                         │
│                  (Messaging API Webhook)                     │
└────────────────────────┬─────────────────────────────────────┘
                         │ HTTPS (固定 IP)
                         │ Webhook POST
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                AWS Cloud (Gateway Service)                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │         Gateway Service (Golang + Gin)                │  │
│  │  - 驗證 LINE webhook 簽章                             │  │
│  │  - 使用者授權白名單檢查                               │  │
│  │  - 確認對話框狀態管理 (PostgreSQL)                    │  │
│  │  - 轉發指令到本地 Mac                                 │  │
│  │  - 審計日誌 (PostgreSQL + 日誌服務)                   │  │
│  │                                                        │  │
│  │  Tailscale IP: 100.64.x.1                             │  │
│  └────────────────────┬───────────────────────────────────┘  │
└───────────────────────┼──────────────────────────────────────┘
                        │
                        │ Tailscale Mesh VPN
                        │ WireGuard 加密隧道
                        │ 自動 NAT 穿透
                        │
┌───────────────────────▼──────────────────────────────────────┐
│              家用網路 (無固定 IP, NAT 後方)                  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │       Mac Control Service (Golang + Gin)              │  │
│  │         Tailscale IP: 100.64.x.2                      │  │
│  │                                                        │  │
│  │  REST API 端點:                                        │  │
│  │  - POST /zoom/join       (加入會議)                   │  │
│  │  - POST /zoom/leave      (離開會議)                   │  │
│  │  - POST /recording/start (開始錄製)                   │  │
│  │  - POST /recording/stop  (停止錄製)                   │  │
│  │  - GET  /status          (查詢狀態)                   │  │
│  │  - GET  /screenshot      (取得截圖)                   │  │
│  │                                                        │  │
│  │  底層實作:                                             │  │
│  │  - osascript (執行 AppleScript 控制 Zoom)             │  │
│  │  - ffmpeg + BlackHole (螢幕錄製 + 系統音訊)           │  │
│  │  - screencapture (macOS 原生截圖工具)                 │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  物理設備: Mac mini / MacBook Pro                            │
└──────────────────────────────────────────────────────────────┘
```

---

### 為什麼選擇 Tailscale？

**Tailscale 技術特性**:
- **Zero Config VPN**: 自動建立 mesh network，無需配置防火牆或路由器
- **WireGuard 協議**: 現代化加密協議，效能優於 OpenVPN
- **NAT 穿透**: 使用 DERP relay 自動穿透家用路由器 NAT
- **跨平台支援**: AWS (Linux)、macOS、Windows、Docker 等全平台支援
- **企業級安全**: 內建 ACL、MagicDNS、設備授權管理

**與其他方案比較**:

| 方案 | 優點 | 缺點 | 評分 |
|------|------|------|------|
| **Tailscale** | ✅ 零配置<br>✅ 企業級安全<br>✅ 穩定性高<br>✅ 免費版足夠 | ⚠️ 依賴第三方服務<br>⚠️ 網路延遲 +10-50ms | ⭐⭐⭐⭐⭐ |
| **ngrok** | ✅ 更簡單（單一工具）<br>✅ 提供 HTTPS | ❌ 免費版連線不穩<br>❌ 付費版 $8+/月 | ⭐⭐⭐⭐ |
| **Cloudflare Tunnel** | ✅ 免費且穩定<br>✅ CDN 加速 | ❌ 僅支援 HTTP/HTTPS<br>❌ 配置較複雜 | ⭐⭐⭐⭐ |
| **WebSocket 反向連接** | ✅ 完全自主控制<br>✅ 無第三方依賴 | ❌ 需自行實作重連<br>❌ 開發複雜度高 | ⭐⭐⭐ |
| **Port Forwarding** | ✅ 最簡單（理論上） | ❌ 安全風險極高<br>❌ 動態 IP 問題<br>❌ 路由器配置困難 | ❌ 不推薦 |

**最終決策**: 採用 **Tailscale**，理由：
1. ✅ 符合憲章 Principle I (Security First)：企業級加密、Zero Trust 模型
2. ✅ 開發效率最高：零配置，專注業務邏輯
3. ✅ 成本可控：免費版支援 20 設備（遠超需求）
4. ✅ 未來擴展性：可輕鬆添加多台 Mac 或多租戶支援

---

## 技術棧選擇

### 後端語言：Golang 1.21+

**選擇理由**:
1. **效能優異**:
   - 編譯型語言，執行效率高
   - 優秀的並發模型（goroutine + channel）
   - 記憶體佔用低（適合 AWS t3.micro 1GB RAM）

2. **部署簡單**:
   - 編譯為單一二進位檔案
   - 無需安裝執行環境（不像 Python 需要虛擬環境）
   - 跨平台編譯（`GOOS=linux GOARCH=amd64 go build`）

3. **生態系統成熟**:
   - 豐富的標準庫（net/http, encoding/json, crypto）
   - 優秀的第三方套件生態
   - 強大的工具鏈（go fmt, go vet, go test）

4. **團隊技能匹配**: 開發者熟悉 Golang

**替代方案考慮**:
- ❌ Python: 執行效率較低，記憶體佔用高
- ❌ Node.js: 單執行緒模型不適合 CPU 密集任務
- ❌ Rust: 學習曲線陡峭，開發速度慢

---

### Web 框架：Gin

**選擇理由**:
1. **高效能**:
   - 基於 httprouter，路由速度極快
   - 記憶體佔用低
   - Benchmark: 處理 40K+ req/sec（比標準庫快 40 倍）

2. **開發體驗好**:
   - API 設計簡潔直觀
   - 內建參數綁定和驗證
   - 豐富的中介軟體生態

3. **功能完整**:
   ```go
   r := gin.Default()
   r.POST("/webhook", validateSignature(), handleWebhook)
   r.GET("/health", healthCheck)
   ```

4. **社群活躍**: GitHub 73k+ stars，文件完善

**替代方案考慮**:
- ❌ Echo: 效能相當，但生態較小
- ❌ Fiber: 基於 fasthttp，API 不夠直觀
- ❌ 標準庫 net/http: 功能太基礎，需自行實作中介軟體

---

### ORM：XORM

**選擇理由**:
1. **輕量級**:
   - 核心程式碼簡潔
   - 學習曲線低
   - 效能開銷小

2. **功能足夠**:
   - SQL Builder 支援複雜查詢
   - Transaction 支援
   - 快取機制
   - 支援 PostgreSQL、MySQL、SQLite

3. **範例**:
   ```go
   type MeetingRecord struct {
       ID          int64     `xorm:"pk autoincr"`
       MeetingID   string    `xorm:"varchar(50) notnull"`
       Password    string    `xorm:"varchar(100)"`
       DisplayName string    `xorm:"varchar(100)"`
       UsedAt      time.Time `xorm:"created"`
   }

   engine.Insert(&record)
   ```

**替代方案考慮**:
- ✅ GORM: 功能更強大（關聯、Hook、Plugin），但較重
  - **結論**: 若未來需要複雜關聯查詢，可考慮遷移到 GORM
- ❌ sqlx: 太輕量，需手寫大量 SQL
- ❌ ent (Facebook): 過於複雜，學習成本高

---

### 資料庫：PostgreSQL 15+

**選擇理由**:
1. **企業級穩定性**:
   - ACID 完整支援
   - 成熟的備份和復原機制
   - 優秀的並發控制

2. **功能豐富**:
   - JSON/JSONB 支援（儲存會議記錄、操作日誌結構化資料）
   - 全文搜尋
   - 豐富的資料類型（timestamp with timezone, UUID）

3. **成本**:
   - 開源免費
   - AWS RDS 提供託管服務（可選）
   - 或在 EC2 自建（成本更低）

4. **範例 Schema**:
   ```sql
   CREATE TABLE meeting_records (
       id SERIAL PRIMARY KEY,
       meeting_id VARCHAR(50) NOT NULL,
       password VARCHAR(100),
       display_name VARCHAR(100),
       metadata JSONB,  -- 儲存額外資訊
       used_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
   );

   CREATE INDEX idx_meeting_used_at ON meeting_records(used_at DESC);
   ```

**替代方案考慮**:
- ❌ MySQL: 功能較少（JSON 支援較弱）
- ❌ SQLite: 不適合多租戶或高並發場景
- ❌ MongoDB: NoSQL 對此場景過度設計

---

## 部署架構

### 雲端部分：AWS

**運算資源**:
- **EC2 執行個體**: t3.micro (1 vCPU, 1GB RAM)
  - 理由：Gateway 服務為輕量級轉發，無重度運算
  - Free Tier：第一年 750 小時/月免費
  - 成本：$7.5/月（超過 Free Tier 後）

- **作業系統**: Ubuntu 22.04 LTS
  - 理由：穩定、安全更新快、Tailscale 支援良好

**網路**:
- **Elastic IP**: 固定 IP 供 LINE Bot webhook 使用
  - 成本：$0（執行個體運行中免費）

- **Tailscale**: 安裝於 EC2 上
  - 取得虛擬 IP: 100.64.x.1

**資料庫選項**:

| 方案 | 成本 | 優點 | 缺點 | 推薦 |
|------|------|------|------|------|
| **EC2 自建 PostgreSQL** | $0 (包含在 t3.micro) | ✅ 成本最低<br>✅ 完全控制 | ⚠️ 需自行維護<br>⚠️ 無自動備份 | ⭐⭐⭐⭐ (初期) |
| **RDS PostgreSQL (db.t3.micro)** | ~$15/月 | ✅ 自動備份<br>✅ 高可用性<br>✅ 免維護 | ❌ 成本較高 | ⭐⭐⭐ (生產環境) |
| **RDS Free Tier** | $0 (12個月) | ✅ 免費 750 小時<br>✅ 20GB 儲存 | ⚠️ 僅限 12 個月 | ⭐⭐⭐⭐⭐ (起步) |

**建議**:
- 初期使用 **EC2 自建** 或 **RDS Free Tier**
- 未來流量增長後遷移到 RDS 託管服務

**安全組設定**:
```
Inbound Rules:
- 443 (HTTPS): 0.0.0.0/0  (LINE webhook)
- 22 (SSH): [你的 IP]      (管理用)

Outbound Rules:
- All traffic: 0.0.0.0/0   (Tailscale + 外部 API)
```

---

### 本地部分：Mac

**硬體需求**:
- **建議配置**: Mac mini (M1/M2) 或 MacBook Pro
- **作業系統**: macOS 13 Ventura 或更新版本
- **記憶體**: 8GB+ (同時運行 Zoom + 錄製)
- **儲存空間**: 至少 100GB 可用空間（錄製檔案）

**軟體環境**:
```bash
# Tailscale VPN 客戶端
brew install tailscale

# ffmpeg (螢幕錄製)
brew install ffmpeg

# BlackHole (系統音訊路由)
brew install blackhole-2ch

# Golang (開發環境)
brew install go@1.21
```

**Control Service 部署**:
```bash
# 編譯 Golang 服務
go build -o mac-control-service main.go

# 設定為系統服務 (launchd)
cp com.linebot.mac-control.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.linebot.mac-control.plist
```

**系統權限設定** (必要):
- ✅ 螢幕錄製權限：系統設定 → 隱私權與安全性 → 螢幕錄製
- ✅ 輔助使用權限：系統設定 → 隱私權與安全性 → 輔助使用
- ✅ 自動化控制：允許終端機控制 Zoom.app

---

## API 設計草稿

### Gateway Service API (面向 LINE Bot)

```
POST /webhook
- 接收 LINE Bot webhook
- 驗證簽章、使用者授權
- 轉發指令到 Mac Control Service
```

### Mac Control Service API (內部 API)

```
POST /zoom/join
Request:
{
  "meeting_id": "1234567890",
  "password": "abc123",
  "display_name": "Allen"
}
Response:
{
  "status": "success|error",
  "message": "已加入會議 1234567890"
}

POST /zoom/leave
Response:
{
  "status": "success|error"
}

POST /recording/start
Response:
{
  "status": "recording",
  "session_id": "rec_20251114_143022"
}

POST /recording/stop
Request:
{
  "session_id": "rec_20251114_143022"
}
Response:
{
  "status": "completed",
  "file_path": "/Users/mac/recordings/rec_20251114_143022.mp4",
  "duration": "00:15:32",
  "file_size": "235MB",
  "download_url": "https://storage.example.com/xxx"
}

GET /status
Response:
{
  "meeting": {
    "status": "in_meeting|idle",
    "meeting_id": "1234567890"
  },
  "recording": {
    "status": "recording|idle",
    "session_id": "rec_xxx",
    "duration": "00:05:23"
  },
  "timestamp": "2025-11-14T14:30:22+08:00"
}

GET /screenshot
Response:
{
  "image_base64": "iVBORw0KGgoAAAANSUhEUgAA..."
}
```

---

## 資料模型草稿

### 會議記錄 (meeting_records)

```sql
CREATE TABLE meeting_records (
    id SERIAL PRIMARY KEY,
    meeting_id VARCHAR(50) NOT NULL,
    password VARCHAR(100),
    display_name VARCHAR(100),
    user_line_id VARCHAR(100) NOT NULL,
    used_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    INDEX idx_user_used_at (user_line_id, used_at DESC)
);
```

### 錄製工作 (recording_sessions)

```sql
CREATE TABLE recording_sessions (
    id SERIAL PRIMARY KEY,
    session_id VARCHAR(100) UNIQUE NOT NULL,
    user_line_id VARCHAR(100) NOT NULL,
    file_path VARCHAR(500),
    file_size BIGINT,
    duration_seconds INT,
    status VARCHAR(20) NOT NULL, -- recording, completed, failed
    started_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    stopped_at TIMESTAMP WITH TIME ZONE,

    INDEX idx_user_started_at (user_line_id, started_at DESC)
);
```

### 使用者操作記錄 (user_action_logs)

```sql
CREATE TABLE user_action_logs (
    id SERIAL PRIMARY KEY,
    user_line_id VARCHAR(100) NOT NULL,
    action_type VARCHAR(50) NOT NULL, -- zoom_join, zoom_leave, recording_start, etc.
    request_payload JSONB,
    response_payload JSONB,
    status VARCHAR(20) NOT NULL, -- success, error
    error_message TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    INDEX idx_user_action_time (user_line_id, created_at DESC)
);
```

### 確認對話框狀態 (confirmation_sessions)

```sql
CREATE TABLE confirmation_sessions (
    id SERIAL PRIMARY KEY,
    session_id VARCHAR(100) UNIQUE NOT NULL,
    user_line_id VARCHAR(100) NOT NULL,
    action_type VARCHAR(50) NOT NULL,
    action_payload JSONB NOT NULL,
    status VARCHAR(20) DEFAULT 'pending', -- pending, confirmed, cancelled, expired
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    INDEX idx_user_pending (user_line_id, status, expires_at)
);
```

---

## 安全性實作策略

### LINE Webhook 簽章驗證

```go
import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/base64"
)

func validateSignature(body []byte, signature string, secret string) bool {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(body)
    expectedSignature := base64.StdEncoding.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(signature), []byte(expectedSignature))
}
```

### 使用者授權白名單

```go
var authorizedUsers = map[string]bool{
    "Uxxxxxxxxxxxxxxxxxxxxxxxxxxxx": true, // Allen's LINE ID
}

func isAuthorized(userID string) bool {
    return authorizedUsers[userID]
}
```

### Tailscale ACL 設定

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["tag:gateway"],
      "dst": ["tag:mac-control:8080"]
    }
  ],
  "tagOwners": {
    "tag:gateway": ["your-email@example.com"],
    "tag:mac-control": ["your-email@example.com"]
  }
}
```

---

## 成本估算

### AWS 雲端成本（月）

| 項目 | 規格 | 費用 |
|------|------|------|
| EC2 t3.micro | 1 vCPU, 1GB RAM | $7.5 (Free Tier 第一年免費) |
| Elastic IP | 1 個固定 IP | $0 (執行中免費) |
| 資料傳輸 | < 1GB/月 | ~$0.09 |
| RDS PostgreSQL (可選) | db.t3.micro | $15 (或 Free Tier 12 個月) |
| **總計** | - | **$7.5~22.5/月** (第一年約 $0) |

### Tailscale 成本

| 方案 | 費用 | 限制 |
|------|------|------|
| Personal (免費) | $0 | 20 設備, 1 使用者 |
| Personal Pro | $6/月 | 100 設備, ACL 進階功能 |

**建議**: 使用**免費版**即可滿足需求

### 本地 Mac 成本

- 硬體成本：已有設備，$0
- 電費：Mac mini 24/7 運行約 $5/月
- 網路費用：家用網路，已包含在現有費用中

**年度總成本估算**: **$90~270** (第一年 Free Tier 約 $60)

---

## 效能目標（技術指標）

| 指標 | 目標值 | 衡量方式 |
|------|--------|----------|
| Gateway API 回應時間 | < 200ms (p95) | Prometheus metrics |
| Mac Control API 回應時間 | < 500ms (p95) | 本地日誌 |
| Tailscale 延遲 | < 50ms | ping 測試 |
| 加入 Zoom 會議時間 | < 30 秒 | 端到端計時 |
| 開始錄製時間 | < 5 秒 | ffmpeg 啟動時間 |
| 截圖生成時間 | < 2 秒 | screencapture 執行時間 |
| 系統可用性 | 99% (排除本地網路斷線) | 監控統計 |

---

## 潛在風險與緩解策略

### 風險 1：Tailscale 服務中斷
- **可能性**: 低（Tailscale 99.9%+ uptime）
- **影響**: 無法控制 Mac
- **緩解**:
  - 監控 Tailscale 連線狀態
  - 文件化備用方案（ngrok 快速切換）

### 風險 2：家用網路斷線
- **可能性**: 中等
- **影響**: 無法控制 Mac
- **緩解**:
  - Gateway 檢測到 Mac 離線時明確通知使用者
  - Mac 設定網路斷線後自動重連

### 風險 3：Mac 系統睡眠
- **可能性**: 高（預設行為）
- **影響**: 無法接收指令
- **緩解**:
  - 設定 Mac 永不睡眠（或僅關閉顯示器）
  - 使用 `caffeinate` 指令防止睡眠

### 風險 4：Zoom 版本更新導致 AppleScript 失效
- **可能性**: 中等
- **影響**: 無法自動化控制 Zoom
- **緩解**:
  - 版本化 AppleScript 腳本
  - 建立測試套件驗證 Zoom 控制
  - 考慮使用 Zoom SDK/API 替代（需 Zoom 企業版）

### 風險 5：錄製檔案儲存空間不足
- **可能性**: 中等
- **影響**: 錄製失敗
- **緩解**:
  - Mac Control Service 啟動前檢查可用空間
  - 空間不足時拒絕開始錄製並通知使用者
  - 定期清理舊錄製檔案

---

## 下一步：整合到 Planning 階段

執行 `/speckit.plan` 時，此文件內容將整合到以下檔案：

1. **plan.md**:
   - Technical Context 章節：Golang, Gin, XORM, PostgreSQL
   - Architecture Design 章節：完整架構圖和說明
   - Constraints 章節：效能目標、成本估算
   - Complexity Tracking 章節：風險緩解

2. **contracts/gateway-api.md**:
   - Gateway Service 的 API 規格

3. **contracts/mac-api.md**:
   - Mac Control Service 的 API 規格

4. **data-model.md**:
   - 完整的資料庫 Schema 設計

5. **deployment.md** (可選):
   - AWS 部署步驟
   - Tailscale 配置指南
   - Mac 環境設定

---

## 附錄：參考資料

- [Tailscale 官方文件](https://tailscale.com/kb/)
- [Gin Framework](https://gin-gonic.com/docs/)
- [XORM 文件](https://xorm.io/)
- [Golang PostgreSQL Driver (pgx)](https://github.com/jackc/pgx)
- [LINE Messaging API](https://developers.line.biz/en/docs/messaging-api/)
- [macOS Automation - AppleScript](https://developer.apple.com/library/archive/documentation/AppleScript/Conceptual/AppleScriptLangGuide/)
- [ffmpeg 文件](https://ffmpeg.org/documentation.html)
