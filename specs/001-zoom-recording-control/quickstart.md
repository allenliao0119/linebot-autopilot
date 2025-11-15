# 快速開始指南

**日期**: 2025-11-15
**階段**: Phase 1 - Design & Contracts
**目標**: 在 30 分鐘內完成本地開發環境設定與功能驗證

## 概述

本指南將引導你完成以下步驟：
1. 安裝開發環境依賴（Golang, PostgreSQL, Tailscale, ffmpeg, BlackHole）
2. 設定 Gateway Service（雲端服務）
3. 設定 Mac Control Service（本地 Mac 服務）
4. 設定 LINE Bot webhook
5. 執行端到端測試

---

## 前置需求

### 硬體需求

| 項目 | 最低需求 | 建議配置 |
|------|----------|----------|
| **Mac 電腦** | macOS 13 Ventura+ | macOS 14 Sonoma+ |
| **記憶體** | 8GB | 16GB |
| **儲存空間** | 50GB 可用空間 | 100GB 可用空間 |
| **網路** | 穩定的家用網路 | 10 Mbps+ 上傳頻寬 |

### 軟體需求

- ✅ Zoom 桌面版應用程式（已安裝）
- ✅ Homebrew 套件管理器
- ✅ Git

---

## 步驟 1: 安裝開發環境依賴

### 1.1 安裝 Golang 1.21+

```bash
# 使用 Homebrew 安裝 Golang
brew install go@1.21

# 驗證安裝
go version
# 預期輸出：go version go1.21.x darwin/arm64
```

### 1.2 安裝 PostgreSQL 15+

```bash
# 安裝 PostgreSQL
brew install postgresql@15

# 啟動 PostgreSQL 服務
brew services start postgresql@15

# 驗證安裝
psql --version
# 預期輸出：psql (PostgreSQL) 15.x

# 建立專案資料庫
createdb linebot_zoom_control

# 測試連線
psql linebot_zoom_control
# 輸入 \q 離開
```

### 1.3 安裝 Tailscale

```bash
# 安裝 Tailscale
brew install tailscale

# 啟動 Tailscale（需登入 Tailscale 帳號）
sudo tailscaled install-system-daemon
tailscale up

# 驗證安裝（查看 Tailscale IP）
tailscale ip -4
# 預期輸出：100.64.x.x
```

**Tailscale 帳號設定**:
1. 訪問 [https://login.tailscale.com](https://login.tailscale.com) 註冊帳號
2. 執行 `tailscale up` 後會開啟瀏覽器進行授權
3. 記錄你的 Tailscale IP（例如：`100.64.12.34`）

### 1.4 安裝 ffmpeg（螢幕錄製）

```bash
# 安裝 ffmpeg
brew install ffmpeg

# 驗證安裝
ffmpeg -version
# 預期輸出：ffmpeg version 6.x
```

### 1.5 安裝 BlackHole（系統音訊路由）

```bash
# 安裝 BlackHole 2ch
brew install blackhole-2ch

# 驗證安裝（開啟音訊 MIDI 設定）
open "/Applications/Utilities/Audio MIDI Setup.app"
```

**設定 Multi-Output Device**:
1. 開啟「音訊 MIDI 設定」應用程式
2. 點擊左下角「+」→「建立多重輸出裝置」
3. 勾選「內建揚聲器」和「BlackHole 2ch」
4. 右鍵點擊新建立的裝置 → 「使用此裝置進行音訊輸出」

---

## 步驟 2: Clone 專案程式碼

```bash
# Clone 專案
git clone https://github.com/allenliao0119/linebot-autopilot.git
cd linebot-autopilot

# 切換到功能分支
git checkout 001-zoom-recording-control
```

---

## 步驟 3: 設定 Gateway Service（雲端服務）

### 3.1 執行資料庫 Migration

```bash
# 進入 Gateway 目錄
cd gateway

# 執行 SQL migration
psql linebot_zoom_control < internal/db/migrations/001_initial_schema.sql

# 驗證表格建立
psql linebot_zoom_control -c "\dt"
# 預期輸出：meeting_records, recording_sessions, user_action_logs, confirmation_sessions
```

### 3.2 設定環境變數

建立 `gateway/.env` 檔案：

```bash
# LINE Bot 設定
LINE_CHANNEL_SECRET=your_line_channel_secret
LINE_CHANNEL_ACCESS_TOKEN=your_line_channel_access_token

# PostgreSQL 設定
DATABASE_URL=postgresql://localhost:5432/linebot_zoom_control?sslmode=disable

# Mac Control Service 位址（Tailscale IP）
MAC_CONTROL_URL=http://100.64.x.x:8080

# 授權使用者白名單（LINE User ID，逗號分隔）
AUTHORIZED_USERS=Uxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 伺服器設定
PORT=8080
```

**取得 LINE Bot 憑證**:
1. 訪問 [LINE Developers Console](https://developers.line.biz/console/)
2. 建立新的 Messaging API Channel
3. 複製「Channel Secret」和「Channel Access Token」

### 3.3 安裝依賴並啟動服務

```bash
# 安裝 Go 模組依賴
go mod download

# 執行測試
go test ./...

# 啟動 Gateway Service
go run cmd/gateway/main.go
```

**預期輸出**:
```
[GIN-debug] Listening and serving HTTP on :8080
Gateway Service started at http://localhost:8080
Connected to PostgreSQL successfully
Tailscale IP: 100.64.x.1
```

---

## 步驟 4: 設定 Mac Control Service（本地 Mac 服務）

### 4.1 設定 macOS 權限

**必要權限設定**:

1. **螢幕錄製權限**:
   - 系統設定 → 隱私權與安全性 → 螢幕錄製
   - 勾選「終端機」或你使用的 IDE（如 VS Code）

2. **輔助使用權限**:
   - 系統設定 → 隱私權與安全性 → 輔助使用
   - 勾選「終端機」或你使用的 IDE

3. **自動化控制**:
   - 首次執行 AppleScript 時會自動詢問，點擊「允許」

### 4.2 設定環境變數

建立 `mac-control/.env` 檔案：

```bash
# 伺服器設定
PORT=8080

# 錄製檔案儲存路徑
RECORDINGS_DIR=/Users/你的使用者名稱/recordings

# ffmpeg 設定
FFMPEG_PRESET=ultrafast
FFMPEG_CRF=23
FFMPEG_FPS=30
```

### 4.3 建立錄製檔案目錄

```bash
# 建立錄製檔案儲存目錄
mkdir -p /Users/你的使用者名稱/recordings
```

### 4.4 安裝依賴並啟動服務

```bash
# 進入 Mac Control 目錄
cd mac-control

# 安裝 Go 模組依賴
go mod download

# 執行測試
go test ./...

# 啟動 Mac Control Service
go run cmd/mac-control/main.go
```

**預期輸出**:
```
[GIN-debug] Listening and serving HTTP on :8080
Mac Control Service started at http://localhost:8080
Tailscale IP: 100.64.x.2
Zoom installed: true
ffmpeg installed: true
BlackHole installed: true
Disk space: 125.5 GB
```

---

## 步驟 5: 設定 LINE Bot Webhook

### 5.1 使用 ngrok 暴露 Gateway Service（開發環境）

```bash
# 安裝 ngrok（若尚未安裝）
brew install ngrok

# 啟動 ngrok（暴露 Gateway Service 的 8080 port）
ngrok http 8080
```

**預期輸出**:
```
Forwarding   https://abc123.ngrok.io -> http://localhost:8080
```

### 5.2 設定 LINE Bot Webhook URL

1. 訪問 [LINE Developers Console](https://developers.line.biz/console/)
2. 選擇你的 Messaging API Channel
3. 前往「Messaging API」頁籤
4. 設定 Webhook URL: `https://abc123.ngrok.io/webhook`
5. 啟用「Use webhook」
6. 點擊「Verify」驗證 webhook（應顯示「Success」）

---

## 步驟 6: 測試完整流程

### 6.1 將 LINE Bot 加入好友

1. 在 LINE Developers Console 複製 Bot 的 QR Code
2. 使用 LINE 掃描 QR Code 加入好友

### 6.2 測試「加入會議」功能

**測試步驟**:
1. 在 LINE 聊天視窗輸入：「加入會議」
2. LINE Bot 應回覆：「請輸入會議 ID」
3. 輸入測試會議 ID（例如：`1234567890`）
4. LINE Bot 應回覆：「請輸入會議密碼（若無密碼請輸入「無」）」
5. 輸入會議密碼或「無」
6. LINE Bot 應回覆：「請輸入顯示名稱」
7. 輸入你的名稱（例如：「Allen」）
8. Mac 應自動啟動 Zoom 並加入會議

**驗證**:
- ✅ Mac 成功啟動 Zoom 應用程式
- ✅ Zoom 自動填入會議 ID 和密碼
- ✅ Zoom 切換為全螢幕模式
- ✅ LINE Bot 回覆：「已成功加入會議 1234567890」

### 6.3 測試「開始錄製」功能

**測試步驟**:
1. 在 LINE 聊天視窗輸入：「開始錄製」
2. LINE Bot 應回覆確認對話框：「確定要開始錄製螢幕嗎？」（附帶「確認」和「取消」按鈕）
3. 點擊「確認」
4. LINE Bot 應回覆：「已開始螢幕錄製（session_id: rec_YYYYMMDD_HHMMSS）」
5. 等待 10 秒
6. 檢查 `/Users/你的使用者名稱/recordings/` 目錄是否有新的 `.mp4` 檔案

**驗證**:
- ✅ 錄製檔案存在且檔案大小持續增長
- ✅ LINE Bot 回覆包含正確的 session_id
- ✅ PostgreSQL `recording_sessions` 表有新記錄（status = 'recording'）

### 6.4 測試「停止錄製」功能

**測試步驟**:
1. 在 LINE 聊天視窗輸入：「停止錄製」
2. LINE Bot 應回覆確認對話框：「確定要停止錄製嗎？」
3. 點擊「確認」
4. LINE Bot 應回覆錄製結果：
   ```
   錄製已完成
   檔名：rec_YYYYMMDD_HHMMSS.mp4
   時長：00:01:23
   檔案大小：45 MB
   ```

**驗證**:
- ✅ 錄製檔案完整且可播放
- ✅ PostgreSQL `recording_sessions` 表記錄更新（status = 'completed'）
- ✅ LINE Bot 回覆包含正確的時長與檔案大小

### 6.5 測試「查看狀態」功能

**測試步驟**:
1. 在 LINE 聊天視窗輸入：「查看狀態」
2. LINE Bot 應回覆：
   ```
   當前狀態：
   會議：進行中（會議 ID: 1234567890）
   錄製：未錄製
   時間：2025-11-15 14:30:22
   ```
3. LINE Bot 應發送螢幕截圖（PNG 圖片）

**驗證**:
- ✅ 狀態資訊正確
- ✅ 螢幕截圖清晰可見

### 6.6 測試「離開會議」功能

**測試步驟**:
1. 在 LINE 聊天視窗輸入：「離開會議」
2. LINE Bot 應回覆確認對話框：「確定要離開會議嗎？」
3. 點擊「確認」
4. Mac 應自動離開 Zoom 會議
5. LINE Bot 應回覆：「已成功離開會議」

**驗證**:
- ✅ Mac 成功離開 Zoom 會議
- ✅ PostgreSQL `user_action_logs` 表有新記錄

---

## 步驟 7: 查看日誌與除錯

### 7.1 查看 Gateway Service 日誌

```bash
# 即時查看日誌（若使用 journalctl）
journalctl -u gateway -f

# 或直接查看終端機輸出
```

### 7.2 查看 Mac Control Service 日誌

```bash
# 即時查看日誌（若使用 launchd）
tail -f ~/Library/Logs/com.linebot.mac-control.log

# 或直接查看終端機輸出
```

### 7.3 查看資料庫記錄

```bash
# 查看最近 10 筆使用者操作記錄
psql linebot_zoom_control -c "
SELECT action_type, status, created_at
FROM user_action_logs
ORDER BY created_at DESC
LIMIT 10;
"

# 查看會議記錄
psql linebot_zoom_control -c "
SELECT meeting_id, display_name, used_at
FROM meeting_records
ORDER BY used_at DESC
LIMIT 5;
"

# 查看錄製記錄
psql linebot_zoom_control -c "
SELECT session_id, status, started_at, file_path
FROM recording_sessions
ORDER BY started_at DESC
LIMIT 5;
"
```

---

## 疑難排解

### 問題 1: Gateway Service 無法啟動

**錯誤訊息**: `dial tcp [::1]:5432: connect: connection refused`

**解決方案**:
```bash
# 檢查 PostgreSQL 是否正在運行
brew services list | grep postgresql

# 若未運行，啟動 PostgreSQL
brew services start postgresql@15
```

---

### 問題 2: Mac Control Service 無法截圖

**錯誤訊息**: `screencapture: Permission denied`

**解決方案**:
1. 系統設定 → 隱私權與安全性 → 螢幕錄製
2. 勾選「終端機」或你使用的 IDE
3. 重新啟動 Mac Control Service

---

### 問題 3: ffmpeg 無法錄製系統音訊

**錯誤訊息**: `Error opening input device BlackHole 2ch`

**解決方案**:
```bash
# 檢查 BlackHole 是否已安裝
ffmpeg -f avfoundation -list_devices true -i ""

# 預期輸出應包含：
# [AVFoundation input device @ ...] [1] BlackHole 2ch

# 若未顯示，重新安裝 BlackHole
brew reinstall blackhole-2ch
```

---

### 問題 4: Tailscale 連線失敗

**錯誤訊息**: `dial tcp 100.64.x.x:8080: i/o timeout`

**解決方案**:
```bash
# 檢查 Tailscale 狀態
tailscale status

# 若顯示離線，重新連線
tailscale up

# 測試 Gateway 與 Mac 的連線
ping $(tailscale ip -4)
```

---

### 問題 5: AppleScript 執行失敗

**錯誤訊息**: `execution error: System Events got an error: osascript is not allowed to send keystrokes`

**解決方案**:
1. 系統設定 → 隱私權與安全性 → 輔助使用
2. 勾選「終端機」或你使用的 IDE
3. 重新執行 AppleScript

---

## 下一步

完成快速開始後，你可以：

1. **部署至 AWS**: 參考 `docs/DEPLOYMENT.md` 進行生產環境部署
2. **建立測試**: 參考 `gateway/tests/` 與 `mac-control/tests/` 撰寫單元測試與整合測試
3. **執行實作**: 使用 `/speckit.tasks` 命令生成任務清單，開始實作功能

---

## 參考資源

- [Golang 官方文件](https://go.dev/doc/)
- [Gin Framework 教學](https://gin-gonic.com/docs/)
- [LINE Messaging API 文件](https://developers.line.biz/en/docs/messaging-api/)
- [Tailscale 快速開始](https://tailscale.com/kb/1017/install/)
- [ffmpeg 螢幕錄製指南](https://trac.ffmpeg.org/wiki/Capture/Desktop)
- [PostgreSQL 教學](https://www.postgresql.org/docs/15/tutorial.html)
