# Implementation Plan: LINE Bot 遠端 Zoom 會議與螢幕錄製控制

**Branch**: `001-zoom-recording-control` | **Date**: 2025-11-15 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-zoom-recording-control/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

本功能透過 LINE Bot 提供遠端控制介面，讓使用者能夠操作位於家用網路環境的 Mac 電腦，自動加入 Zoom 會議並進行螢幕錄製。系統採用雲地混合架構（Cloud Gateway + Local Agent），透過 Tailscale VPN 建立安全的加密通訊隧道，解決家用網路無固定 IP 與 NAT 穿透的問題。

**核心技術方案**:
- **架構**: AWS Gateway (Golang + Gin) ↔ Tailscale VPN ↔ Mac Agent (Golang + Gin)
- **Zoom 自動化**: AppleScript 控制 Zoom 應用程式視窗與操作
- **螢幕錄製**: ffmpeg + BlackHole 音訊路由實現螢幕與系統音訊同步錄製
- **資料儲存**: PostgreSQL (雲端審計日誌) + Local filesystem (錄製檔案)
- **安全性**: LINE webhook 簽章驗證、使用者白名單、Tailscale ACL、操作確認機制

## Technical Context

**Language/Version**: Golang 1.21+
**Primary Dependencies**:
- Gateway Service: Gin (web framework), XORM (ORM), @line/bot-sdk-go, crypto/hmac (簽章驗證)
- Mac Control Service: Gin (web framework), os/exec (執行 AppleScript/ffmpeg), net/http (HTTP client)
- Shared: Tailscale (VPN client), PostgreSQL driver (pgx)

**Storage**:
- PostgreSQL 15+ (雲端): 會議記錄、錄製工作、使用者操作日誌、確認對話框狀態
- Local filesystem (Mac): 錄製影片檔案 (.mp4)
- Environment variables: API keys、Tailscale auth key、使用者白名單

**Testing**:
- Unit tests: Go testing package + testify assertions
- Integration tests: Testcontainers (PostgreSQL), httptest (API testing)
- Contract tests: LINE Bot webhook mock, Mac Control API mock
- E2E tests: 手動測試實際 Zoom 加入與錄製流程

**Target Platform**:
- Gateway: AWS EC2 t3.micro (Ubuntu 22.04 LTS, 1 vCPU, 1GB RAM)
- Mac Control: macOS 13 Ventura+ (支援 AppleScript, ffmpeg, screen recording permissions)

**Project Type**: Web application (分散式後端服務，雲地混合架構)

**Performance Goals**:
- Gateway API 回應時間: < 200ms (p95)
- Mac Control API 回應時間: < 500ms (p95)
- 加入 Zoom 會議總時長: < 30 秒（從 LINE 點擊到 Mac 顯示會議畫面）
- 開始錄製延遲: < 5 秒（從點擊按鈕到 ffmpeg 開始錄製）
- 截圖生成時間: < 2 秒
- Tailscale VPN 延遲: < 50ms (ping Gateway ↔ Mac)

**Constraints**:
- 單一使用者控制模式：同一時間僅允許一個 LINE 使用者操作 Mac
- macOS 專屬：Mac Control Service 僅支援 macOS (依賴 AppleScript、screencapture)
- 需穩定網路連線：家用網路斷線期間無法接收遠端指令
- 系統權限要求：Mac 需授予螢幕錄製、輔助使用、自動化控制權限
- 錄製儲存空間：至少 100GB 可用空間（單次會議錄製約 1-5GB）
- Tailscale 免費版限制：最多 20 台裝置（遠超需求，但需注意）

**Scale/Scope**:
- 單一 Mac 電腦控制
- 1-3 個授權 LINE 使用者
- 預期每日操作次數: < 50 次（加入會議、錄製等操作）
- 會議記錄保留: 最近 5 筆
- 審計日誌保留: 30 天（可配置）

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### ✅ Principle I: Security First (NON-NEGOTIABLE)

- ✅ **明確使用者授權**: 所有操作透過 LINE Bot 確認對話框（FR-010, FR-011, FR-024）
- ✅ **輸入驗證與防注入**: 會議 ID 限定純數字格式，密碼透過環境變數傳遞（避免 shell injection）
- ✅ **安全憑證儲存**: LINE Bot token、Tailscale auth key 儲存於環境變數，絕不硬編碼（FR-029）
- ✅ **安全通訊**:
  - LINE webhook 使用 HTTPS + HMAC-SHA256 簽章驗證（FR-026）
  - Tailscale 提供 WireGuard 加密隧道（Gateway ↔ Mac）
- ✅ **審計日誌**: 所有操作記錄至 PostgreSQL user_action_logs 表（FR-028）
- ✅ **安全失敗**: webhook 簽章驗證失敗或使用者未授權時直接拒絕請求（HTTP 403）

**實作檢查點**:
- [ ] LINE webhook 簽章驗證中介軟體（`validateSignature()`）
- [ ] 使用者白名單授權檢查（`isAuthorized(userID)`）
- [ ] 環境變數驗證啟動檢查（FR-030）
- [ ] Tailscale ACL 設定限制 Gateway → Mac 通訊

### ✅ Principle II: User Confirmation for Critical Actions

- ✅ **確認對話框**: 開始錄製、停止錄製、離開會議皆需確認（FR-010, FR-011, FR-024）
- ✅ **明確選項**: LINE Bot 提供「確認/取消」按鈕（Quick Reply Buttons）
- ✅ **超時機制**: 60 秒未回應自動取消操作（FR-012, FR-025）
- ✅ **狀態維持**: 使用者拒絕或超時時系統狀態不變（FR-035）
- ✅ **操作回饋**: 每次操作完成後發送狀態通知訊息

**實作檢查點**:
- [ ] PostgreSQL confirmation_sessions 表（管理確認狀態）
- [ ] 超時自動清理機制（cron job 或 background worker）
- [ ] LINE Bot Quick Reply 按鈕實作

### ✅ Principle III: Robust Error Handling

- ✅ **異常捕獲**: 所有 API handler 使用 defer/recover 避免 panic 導致服務崩潰（FR-031）
- ✅ **使用者友善訊息**: 錯誤訊息避免技術術語（FR-032, FR-008）
  - 範例：「會議 ID 或密碼錯誤，請重新檢查」而非「AppleScript execution failed: error -1719」
- ✅ **詳細日誌**: 錯誤包含堆疊追蹤、系統狀態記錄至日誌（FR-033）
- ✅ **重試機制**: Tailscale VPN 連線失敗時使用指數退避重試（FR-034）
- ✅ **一致性保證**: 錄製失敗時確保 recording_sessions 狀態更新為 "failed"（FR-035）

**實作檢查點**:
- [ ] Global error handler middleware (Gin recovery)
- [ ] 結構化日誌（使用 logrus 或 zap）
- [ ] 重試策略實作（使用 exponential backoff library）
- [ ] 錯誤訊息映射表（technical error → user-friendly message）

### ✅ Principle IV: Status Visibility

- ✅ **狀態查詢**: 使用者可隨時查詢會議狀態、錄製狀態、螢幕截圖（FR-018, FR-019, FR-020, FR-021）
- ✅ **主動通知**: 加入會議成功、錄製開始/停止時主動發送 LINE 訊息
- ✅ **時間戳記**: 所有狀態資訊包含 timestamp（FR-022）
- ✅ **視覺確認**: 提供螢幕截圖功能（FR-019）
- ✅ **審計日誌**: user_action_logs 表記錄所有狀態轉換

**實作檢查點**:
- [ ] GET /status API endpoint
- [ ] GET /screenshot API endpoint (macOS screencapture)
- [ ] LINE Bot 主動推送訊息機制
- [ ] 所有 response payload 包含 timestamp 欄位

### ✅ Principle V: Testability and Integration Testing

- ✅ **測試優先**: 先撰寫測試再實作（TDD 方法論）
- ✅ **整合測試**: 測試 LINE Bot webhook、Zoom 控制、錄製流程（定義於 Testing 章節）
- ✅ **契約測試**: Mock LINE Bot API、Mock Mac Control API
- ✅ **端到端測試**: 手動測試完整使用者旅程（加入會議 → 錄製 → 停止 → 離開）
- ✅ **Mock 外部依賴**: 單元測試使用 httptest mock HTTP calls

**實作檢查點**:
- [ ] Unit tests 覆蓋率 > 80%
- [ ] Integration tests (使用 Testcontainers 啟動 PostgreSQL)
- [ ] Contract tests (LINE webhook payload validation)
- [ ] E2E test checklist 文件

### ✅ Principle VI: Platform-Specific Reliability

- ✅ **穩定 API**: 優先使用 macOS 原生工具（screencapture, osascript）
- ✅ **權限處理**: 啟動時檢查螢幕錄製、輔助使用權限（FR-030）
- ✅ **優雅降級**: 權限缺失時回傳明確錯誤訊息並提供設定指引（FR-032）
- ✅ **版本測試**: 支援 macOS 13 Ventura 和 macOS 14 Sonoma（定義於 Target Platform）
- ✅ **文件化**: 建立系統需求與設定步驟文件（quickstart.md）
- ✅ **邊界處理**: 處理多螢幕、全螢幕模式、視窗焦點等 macOS 特性（FR-005）

**實作檢查點**:
- [ ] macOS 權限檢查函式（使用 tccutil 或 AppleScript 檢測）
- [ ] 多螢幕環境測試
- [ ] Zoom 全螢幕模式切換測試
- [ ] macOS 版本相容性測試矩陣

### ✅ Principle VII: Documentation Language (NON-NEGOTIABLE)

- ✅ **規格文件**: spec.md 完全使用繁體中文撰寫
- ✅ **實作計畫**: plan.md (本文件) 使用繁體中文
- ✅ **錯誤訊息**: LINE Bot 回傳的使用者訊息使用繁體中文（FR-032）
- ✅ **狀態訊息**: 會議加入成功、錄製開始等通知使用繁體中文
- ✅ **程式碼例外**: 變數名稱、函式名稱保持英文（例如 `validateSignature`, `handleWebhook`）
- ✅ **技術術語**: 可保留英文但需中文說明（例如「Tailscale (VPN 服務)」）

**實作檢查點**:
- [ ] LINE Bot 訊息模板檔案（messages.go 或 messages.yaml）使用繁體中文
- [ ] 錯誤訊息對照表（error_messages_zh_tw.go）
- [ ] 複雜邏輯的中文註解（例如 AppleScript 腳本說明）
- [ ] README.md 和 quickstart.md 使用繁體中文

### 🟡 Complexity Tracking

無違反憲章的複雜度引入，無需額外說明。

**決策總結**: 本功能設計完全符合憲章所有核心原則，無需妥協或例外處理。

## Project Structure

### Documentation (this feature)

```text
specs/001-zoom-recording-control/
├── spec.md              # 功能規格（已完成）
├── tech-decisions.md    # 技術決策記錄（已完成）
├── plan.md              # 本檔案 (/speckit.plan 輸出)
├── research.md          # Phase 0 輸出 (技術研究與最佳實踐)
├── data-model.md        # Phase 1 輸出 (資料庫 Schema)
├── quickstart.md        # Phase 1 輸出 (快速開始指南)
├── contracts/           # Phase 1 輸出 (API 契約)
│   ├── gateway-api.yaml # Gateway Service OpenAPI 規格
│   └── mac-api.yaml     # Mac Control Service OpenAPI 規格
├── checklists/          # 檢查清單
│   └── requirements.md  # 需求檢查清單（已完成）
└── tasks.md             # Phase 2 輸出 (/speckit.tasks - 尚未建立)
```

### Source Code (repository root)

本專案採用**分散式雲地混合架構**，包含兩個獨立的 Golang 服務：

```text
gateway/                 # AWS 雲端 Gateway Service
├── cmd/
│   └── gateway/
│       └── main.go      # 服務入口點
├── internal/
│   ├── handlers/        # HTTP handlers
│   │   ├── webhook.go   # LINE Bot webhook handler
│   │   └── health.go    # 健康檢查
│   ├── middleware/      # HTTP middleware
│   │   ├── auth.go      # 使用者白名單驗證
│   │   └── signature.go # LINE webhook 簽章驗證
│   ├── services/        # 業務邏輯
│   │   ├── confirmation.go  # 確認對話框管理
│   │   ├── mac_client.go    # Mac Control API 客戶端
│   │   └── line_client.go   # LINE Bot API 客戶端
│   ├── models/          # 資料模型
│   │   ├── meeting.go
│   │   ├── recording.go
│   │   └── audit_log.go
│   └── db/              # 資料庫層
│       ├── postgres.go  # PostgreSQL 連線
│       └── migrations/  # SQL migrations
├── tests/
│   ├── unit/            # 單元測試
│   ├── integration/     # 整合測試 (Testcontainers)
│   └── contract/        # 契約測試 (LINE webhook mock)
├── configs/
│   └── config.yaml      # 設定檔範例
├── go.mod
├── go.sum
└── Dockerfile           # AWS 部署用

mac-control/             # 本地 Mac Control Service
├── cmd/
│   └── mac-control/
│       └── main.go      # 服務入口點
├── internal/
│   ├── handlers/        # HTTP handlers
│   │   ├── zoom.go      # Zoom 控制 API (/zoom/join, /zoom/leave)
│   │   ├── recording.go # 錄製控制 API (/recording/start, /recording/stop)
│   │   └── status.go    # 狀態查詢 API (/status, /screenshot)
│   ├── automation/      # macOS 自動化層
│   │   ├── zoom.go      # Zoom AppleScript 控制
│   │   ├── ffmpeg.go    # ffmpeg 螢幕錄製
│   │   └── screenshot.go # macOS screencapture
│   ├── models/          # 資料模型
│   │   └── status.go
│   └── permissions/     # macOS 權限檢查
│       └── check.go
├── scripts/
│   ├── zoom/            # AppleScript 腳本
│   │   ├── join.scpt    # 加入會議
│   │   ├── leave.scpt   # 離開會議
│   │   └── fullscreen.scpt # 切換全螢幕
│   └── setup/           # 環境設定腳本
│       └── install_deps.sh # 安裝 ffmpeg, BlackHole
├── tests/
│   ├── unit/
│   └── integration/     # 實際 Zoom 測試
├── configs/
│   └── config.yaml
├── recordings/          # 錄製檔案儲存目錄（gitignore）
├── go.mod
├── go.sum
└── com.linebot.mac-control.plist # macOS launchd 服務定義

shared/                  # 共用程式碼（可選，未來擴展用）
└── types/               # 共用型別定義
    └── api.go           # API request/response 結構

docs/                    # 專案級文件
├── README.md            # 專案總覽（繁體中文）
├── ARCHITECTURE.md      # 架構說明
├── DEPLOYMENT.md        # 部署指南
│   ├── aws-gateway.md   # AWS 部署步驟
│   ├── tailscale.md     # Tailscale 設定
│   └── mac-setup.md     # Mac 環境設定
└── API.md               # API 文件總覽

.github/
└── workflows/
    ├── gateway-ci.yml   # Gateway CI/CD
    └── mac-control-ci.yml # Mac Control CI/CD

scripts/                 # 專案級腳本
├── setup-dev.sh         # 開發環境設定
└── deploy.sh            # 部署腳本

.env.example             # 環境變數範例
.gitignore
Makefile                 # 統一建構指令
```

**Structure Decision**:

本專案選擇**微服務架構**（兩個獨立的 Golang 服務），而非單體式專案，理由如下：

1. **部署環境隔離**: Gateway 部署於 AWS 雲端，Mac Control 部署於本地 Mac，具有不同的執行環境和依賴
2. **技術棧差異**: Mac Control 依賴 macOS 專屬 API (AppleScript, screencapture)，無法跨平台執行
3. **獨立開發與測試**: 兩個服務可獨立開發、測試、部署，降低耦合度
4. **擴展性**: 未來可輕鬆擴展為多租戶架構（一個 Gateway 對接多個 Mac Control）

每個服務遵循 **標準 Golang 專案佈局** ([golang-standards/project-layout](https://github.com/golang-standards/project-layout)):
- `cmd/`: 應用程式入口點
- `internal/`: 私有程式碼，不可被外部匯入
- `tests/`: 測試檔案（與 `*_test.go` 並行使用）
- `configs/`: 設定檔範例

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

無憲章違反項目，本節留空。
