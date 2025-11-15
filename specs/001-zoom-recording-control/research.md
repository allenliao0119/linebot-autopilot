# 技術研究報告：LINE Bot 遠端 Zoom 控制

**日期**: 2025-11-15
**階段**: Phase 0 - Research & Outline
**相關文件**: [tech-decisions.md](./tech-decisions.md), [plan.md](./plan.md)

## 研究摘要

本文件記錄為實現 LINE Bot 遠端 Zoom 會議控制功能所進行的技術研究與決策過程。主要研究領域包含：系統架構設計、程式語言選擇、通訊協定、自動化技術、錄製方案等。

## 1. 系統架構

### 核心問題

**挑戰**: 本地 Mac 電腦位於家用網路環境（動態 IP、NAT 後方），而 LINE Bot webhook 需要固定 IP 的公網端點。

**研究問題**:
- 如何讓 LINE Bot 安全地控制位於私有網路的 Mac？
- 如何避免複雜的路由器設定（port forwarding）？
- 如何確保通訊安全且穩定？

### 決策：雲地混合架構 + Tailscale VPN

**選擇方案**: AWS Gateway (雲端) ↔ Tailscale VPN ↔ Mac Control Service (本地)

**架構流程**:
```
LINE Platform (webhook)
  → AWS Gateway (固定 IP, 公網可達)
  → Tailscale VPN 隧道 (WireGuard 加密)
  → Mac Control Service (家用網路, 100.x.x.x 虛擬 IP)
```

**選擇理由**:
1. **零配置 VPN**: Tailscale 自動建立 mesh network，無需設定防火牆或 NAT 穿透
2. **企業級安全**: WireGuard 加密協定 + ACL 存取控制，符合憲章 Principle I (Security First)
3. **穩定性**: Tailscale 提供 99.9%+ uptime，DERP relay 確保連線穩定
4. **開發效率**: 專注業務邏輯，不需處理複雜網路配置
5. **成本可控**: 免費版支援 20 設備（遠超本專案需求）

**替代方案評估**:

| 方案 | 優點 | 缺點 | 不選擇的原因 |
|------|------|------|-------------|
| **ngrok** | 簡單易用、提供 HTTPS | 免費版連線不穩、付費版 $8+/月 | 成本較高、免費版不適合生產環境 |
| **Cloudflare Tunnel** | 免費且穩定、CDN 加速 | 僅支援 HTTP/HTTPS、配置較複雜 | 功能限制（僅 HTTP），不支援任意 TCP |
| **WebSocket 反向連接** | 完全自主控制、無第三方依賴 | 需自行實作重連機制、開發複雜度高 | 開發時間成本高，可靠性需自行保證 |
| **Port Forwarding** | 理論上最簡單 | 安全風險極高、動態 IP 問題、路由器配置困難 | 違反憲章安全原則、不可行 |

**參考資料**:
- [Tailscale 官方文件](https://tailscale.com/kb/)
- [WireGuard 協定白皮書](https://www.wireguard.com/papers/wireguard.pdf)

---

## 2. 程式語言

### 決策：Golang 1.21+

**選擇理由**:

1. **效能優異**:
   - 編譯型語言，執行效率高於直譯語言（Python、Node.js）
   - 優秀的並發模型（goroutine），適合處理多個 webhook 請求
   - 記憶體佔用低，適合 AWS t3.micro (1GB RAM) 執行

2. **部署簡單**:
   - 編譯為單一二進位檔案，無執行環境依賴
   - 跨平台編譯支援（`GOOS=linux GOARCH=amd64 go build`）
   - Docker 容器映像檔小（使用 multi-stage build < 20MB）

3. **生態系統成熟**:
   - 豐富的標準庫（`net/http`, `encoding/json`, `crypto/hmac`）
   - 優秀的第三方套件（Gin, XORM, Testify）
   - 強大的工具鏈（go fmt, go vet, go test）

4. **團隊技能匹配**: 開發者熟悉 Golang 語法與慣例

**替代方案評估**:

| 語言 | 優點 | 缺點 | 不選擇的原因 |
|------|------|------|-------------|
| **Python** | 生態系統最豐富、開發速度快 | 執行效率低、記憶體佔用高、需虛擬環境 | 效能不符合 < 200ms API 回應要求 |
| **Node.js** | 非同步 I/O 適合 webhook、生態豐富 | 單執行緒模型、CPU 密集任務效能差 | 不適合處理 ffmpeg 等 CPU 密集操作 |
| **Rust** | 效能最優、記憶體安全 | 學習曲線陡峭、開發速度慢、生態較小 | 開發時間成本過高 |

**參考資料**:
- [Golang 官方文件](https://go.dev/doc/)
- [Golang Performance Benchmarks](https://benchmarksgame-team.pages.debian.net/benchmarksgame/)

---

## 3. Web 框架

### 決策：Gin

**選擇理由**:

1. **高效能**:
   - 基於 httprouter，路由速度極快
   - Benchmark: 40K+ req/sec（比標準庫 `net/http` 快 40 倍）
   - 記憶體佔用低

2. **開發體驗優秀**:
   - API 設計簡潔直觀
   - 內建 JSON 綁定與驗證
   - 豐富的中介軟體生態（CORS, Logger, Recovery）

3. **社群活躍**:
   - GitHub 73k+ stars
   - 文件完善、範例豐富
   - 大量生產環境實踐

**程式碼範例**:
```go
r := gin.Default()

// 中介軟體：簽章驗證
r.Use(middleware.ValidateSignature())

// 路由定義
r.POST("/webhook", handlers.HandleWebhook)
r.GET("/health", handlers.HealthCheck)

r.Run(":8080")
```

**替代方案評估**:

| 框架 | 優點 | 缺點 | 不選擇的原因 |
|------|------|------|-------------|
| **Echo** | 效能相當、API 相似 | 社群較小、中介軟體生態較少 | Gin 社群更活躍，資源更豐富 |
| **Fiber** | 效能最高（基於 fasthttp） | API 不直觀、與標準庫不相容 | 學習成本高、生態不成熟 |
| **net/http (標準庫)** | 無第三方依賴、最穩定 | 功能基礎、需自行實作中介軟體 | 開發效率低 |

**參考資料**:
- [Gin Framework 官方文件](https://gin-gonic.com/docs/)
- [Gin GitHub Repository](https://github.com/gin-gonic/gin)

---

## 4. ORM

### 決策：XORM

**選擇理由**:

1. **輕量級**: 核心程式碼簡潔，學習曲線低
2. **功能足夠**: SQL Builder、Transaction、快取機制、多資料庫支援
3. **效能優秀**: 相比 GORM 更輕量，記憶體佔用更低

**程式碼範例**:
```go
type MeetingRecord struct {
    ID          int64     `xorm:"pk autoincr"`
    MeetingID   string    `xorm:"varchar(50) notnull"`
    Password    string    `xorm:"varchar(100)"`
    DisplayName string    `xorm:"varchar(100)"`
    UserLineID  string    `xorm:"varchar(100) notnull"`
    UsedAt      time.Time `xorm:"created"`
}

// 插入記錄
engine.Insert(&record)

// 查詢最近 5 筆
var records []MeetingRecord
engine.Where("user_line_id = ?", userID).
       OrderBy("used_at DESC").
       Limit(5).
       Find(&records)
```

**替代方案評估**:

| ORM | 優點 | 缺點 | 評估結果 |
|-----|------|------|----------|
| **GORM** | 功能最強大（關聯、Hook、Plugin） | 較重量、效能稍低 | 未來若需複雜關聯可遷移 |
| **sqlx** | 最輕量、貼近原生 SQL | 需手寫大量 SQL、無 Schema 管理 | 開發效率低 |
| **ent (Facebook)** | 型別安全、Code Generation | 學習曲線陡峭、過於複雜 | 不適合小型專案 |

**決策**: 初期使用 XORM，若未來需求增長（例如多對多關聯查詢），可評估遷移至 GORM。

**參考資料**:
- [XORM 官方文件](https://xorm.io/)
- [GORM vs XORM 比較](https://github.com/go-xorm/xorm/issues/1457)

---

## 5. 資料庫

### 決策：PostgreSQL 15+

**選擇理由**:

1. **企業級穩定性**:
   - ACID 完整支援
   - 成熟的備份與復原機制（pg_dump, PITR）
   - 優秀的並發控制（MVCC）

2. **功能豐富**:
   - **JSONB 支援**: 儲存會議記錄 metadata、操作日誌 payload（結構化與彈性兼具）
   - **Timestamp with timezone**: 精確記錄跨時區操作時間
   - **全文搜尋**: 未來可支援審計日誌搜尋

3. **成本可控**:
   - 開源免費
   - AWS RDS Free Tier: 12 個月免費（db.t3.micro, 20GB 儲存）
   - 或在 EC2 自建（成本更低）

**Schema 範例**:
```sql
CREATE TABLE meeting_records (
    id SERIAL PRIMARY KEY,
    meeting_id VARCHAR(50) NOT NULL,
    password VARCHAR(100),
    display_name VARCHAR(100),
    user_line_id VARCHAR(100) NOT NULL,
    metadata JSONB,  -- 儲存額外資訊（例如會議主題）
    used_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    INDEX idx_user_used_at (user_line_id, used_at DESC)
);
```

**替代方案評估**:

| 資料庫 | 優點 | 缺點 | 不選擇的原因 |
|--------|------|------|-------------|
| **MySQL** | 廣泛使用、社群大 | JSON 支援較弱、功能較少 | PostgreSQL 功能更豐富 |
| **SQLite** | 零配置、檔案型資料庫 | 不適合並發寫入、無網路存取 | 不適合多租戶或高並發場景 |
| **MongoDB** | NoSQL 彈性高 | 無 ACID、過度設計 | 本專案資料結構明確，不需 NoSQL |

**參考資料**:
- [PostgreSQL 官方文件](https://www.postgresql.org/docs/)
- [AWS RDS PostgreSQL](https://aws.amazon.com/rds/postgresql/)

---

## 6. Zoom 自動化

### 決策：AppleScript

**選擇理由**:

1. **macOS 原生支援**: 無需額外安裝工具
2. **穩定性高**: Apple 官方維護，API 變動少
3. **功能完整**: 可控制視窗、點擊按鈕、輸入文字
4. **社群範例豐富**: 大量現成的 Zoom 控制腳本

**程式碼範例** (加入會議):
```applescript
tell application "zoom.us"
    activate

    -- 點擊「加入會議」按鈕
    tell application "System Events"
        tell process "zoom.us"
            click button "Join a Meeting" of window 1
            delay 1

            -- 輸入會議 ID
            keystroke "1234567890"
            delay 0.5

            -- 點擊加入
            click button "Join" of window 1
        end tell
    end tell
end tell
```

**Golang 整合** (執行 AppleScript):
```go
func JoinZoomMeeting(meetingID, password, displayName string) error {
    script := fmt.Sprintf(`osascript zoom/join.scpt %s %s %s`,
                          meetingID, password, displayName)
    cmd := exec.Command("sh", "-c", script)
    output, err := cmd.CombinedOutput()
    if err != nil {
        return fmt.Errorf("AppleScript failed: %s, %v", output, err)
    }
    return nil
}
```

**替代方案評估**:

| 方案 | 優點 | 缺點 | 不選擇的原因 |
|------|------|------|-------------|
| **Zoom SDK** | 官方支援、功能最完整 | 需 Zoom 企業版、開發複雜度高 | 成本過高（企業版 $20/月/人） |
| **PyAutoGUI (Python)** | 跨平台、易用 | 依賴螢幕座標、不穩定 | 視窗大小變化會導致失敗 |
| **Selenium WebDriver** | 控制 Zoom Web 版 | Web 版功能受限、效能差 | 無法控制桌面版的全螢幕模式 |

**風險緩解**:
- **Zoom 版本更新導致 UI 變化**: 建立測試套件驗證腳本，版本化管理 AppleScript
- **未來考慮**: 若 Zoom 大幅改版，可評估 Zoom SDK（需升級企業版）

**參考資料**:
- [macOS Automation - AppleScript](https://developer.apple.com/library/archive/documentation/AppleScript/Conceptual/AppleScriptLangGuide/)
- [Zoom AppleScript 範例](https://github.com/search?q=zoom+applescript)

---

## 7. 螢幕錄製

### 決策：ffmpeg + BlackHole 音訊路由

**選擇理由**:

1. **功能完整**:
   - 同時錄製螢幕與系統音訊
   - 支援多種影片格式（MP4, MOV, WebM）
   - 可調整解析度、位元率、FPS

2. **開源免費**: MIT License，無授權費用

3. **跨平台**: 未來若支援 Windows/Linux，ffmpeg 可複用

4. **社群成熟**: Stack Overflow、GitHub 有大量範例

**技術架構**:
```
系統音訊輸出 → BlackHole (虛擬音訊裝置) → ffmpeg 輸入
螢幕畫面     → ffmpeg 螢幕捕捉                → MP4 檔案
```

**程式碼範例**:
```bash
# macOS 螢幕錄製指令（含系統音訊）
ffmpeg -f avfoundation \
       -i "1:BlackHole 2ch" \
       -r 30 \
       -vcodec libx264 \
       -acodec aac \
       -preset ultrafast \
       -crf 23 \
       output.mp4
```

**Golang 整合**:
```go
func StartRecording(outputPath string) (*exec.Cmd, error) {
    args := []string{
        "-f", "avfoundation",
        "-i", "1:BlackHole 2ch",  // 螢幕:音訊裝置
        "-r", "30",
        "-vcodec", "libx264",
        "-acodec", "aac",
        "-preset", "ultrafast",
        outputPath,
    }
    cmd := exec.Command("ffmpeg", args...)
    err := cmd.Start()
    if err != nil {
        return nil, err
    }
    return cmd, nil  // 回傳 cmd 物件供後續停止錄製
}

func StopRecording(cmd *exec.Cmd) error {
    return cmd.Process.Signal(os.Interrupt)  // 發送 Ctrl+C 訊號
}
```

**替代方案評估**:

| 方案 | 優點 | 缺點 | 不選擇的原因 |
|------|------|------|-------------|
| **QuickTime Player** | macOS 原生、UI 簡單 | 無 API、無法程式化控制 | 無法遠端自動化 |
| **ScreenFlow** | 專業錄製軟體、功能強大 | 付費軟體 ($169)、無 CLI | 成本高、無自動化 API |
| **Zoom 內建錄製** | 最簡單、無需額外工具 | 違反需求（FR-009 明確要求使用第三方工具） | 需求明確排除 |

**BlackHole 設定**:
```bash
# 安裝 BlackHole（虛擬音訊裝置）
brew install blackhole-2ch

# 設定音訊路由（系統設定 → 聲音 → 輸出裝置）
# 建立 Multi-Output Device:
# - 內建揚聲器（讓使用者聽到聲音）
# - BlackHole 2ch（讓 ffmpeg 錄製聲音）
```

**參考資料**:
- [ffmpeg 官方文件](https://ffmpeg.org/documentation.html)
- [BlackHole Audio Driver](https://github.com/ExistentialAudio/BlackHole)
- [ffmpeg macOS Screen Recording Guide](https://trac.ffmpeg.org/wiki/Capture/Desktop)

---

## 8. 部署平台

### 決策：AWS EC2 t3.micro (Gateway) + 本地 Mac (Control Service)

**Gateway Service (AWS)**:

**選擇理由**:
1. **固定 IP**: Elastic IP 提供固定公網 IP 供 LINE Bot webhook 使用
2. **成本效益**: t3.micro (1 vCPU, 1GB RAM) Free Tier 第一年免費，超過後 $7.5/月
3. **生態成熟**: AWS RDS、CloudWatch、S3 等服務整合方便
4. **可靠性**: 99.99% SLA

**配置**:
- **執行個體**: t3.micro (1 vCPU, 1GB RAM)
- **作業系統**: Ubuntu 22.04 LTS
- **Elastic IP**: 固定 IP
- **安全組**:
  - Inbound: 443 (HTTPS), 22 (SSH 限制 IP)
  - Outbound: All traffic (Tailscale + 外部 API)

**成本估算**:
| 項目 | 費用 |
|------|------|
| EC2 t3.micro | $0 (Free Tier 12 個月) / $7.5/月 (超過後) |
| Elastic IP | $0 (執行中) |
| 資料傳輸 | ~$0.09/月 (< 1GB) |
| **總計** | **$0~7.5/月** |

**替代方案評估**:

| 平台 | 優點 | 缺點 | 不選擇的原因 |
|------|------|------|-------------|
| **AWS Lambda** | Serverless、按需計費 | 冷啟動延遲、連線限制 | Tailscale VPN 需持續連線，不適合 Lambda |
| **Google Cloud Run** | 自動擴展、容器化 | 同 Lambda，不適合長連線 | 同上 |
| **DigitalOcean Droplet** | 價格更低 ($6/月) | 生態較小、無 Free Tier | 初期使用 AWS Free Tier 成本更低 |
| **Heroku** | 部署簡單 | 免費版已取消、付費版 $7/月 | 無明顯優勢 |

**Mac Control Service (本地)**:

**部署方式**: macOS launchd 系統服務

```xml
<!-- com.linebot.mac-control.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.linebot.mac-control</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/mac-control-service</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

**安裝指令**:
```bash
cp com.linebot.mac-control.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.linebot.mac-control.plist
```

**參考資料**:
- [AWS EC2 定價](https://aws.amazon.com/ec2/pricing/)
- [AWS Free Tier](https://aws.amazon.com/free/)
- [macOS launchd 文件](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)

---

## 9. LINE Bot 整合

### 決策：LINE Messaging API + line-bot-sdk-go

**選擇理由**:
1. **官方 SDK**: LINE 官方維護，API 變動時及時更新
2. **功能完整**: 支援 webhook、push message、rich menu、quick reply
3. **文件完善**: 豐富的範例與說明

**webhook 簽章驗證**:
```go
import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/base64"
)

func ValidateSignature(body []byte, signature string, secret string) bool {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(body)
    expected := base64.StdEncoding.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(signature), []byte(expected))
}
```

**Quick Reply 按鈕範例** (確認對話框):
```go
import "github.com/line/line-bot-sdk-go/v7/linebot"

func SendConfirmation(bot *linebot.Client, userID string, message string) error {
    quickReplyItems := linebot.NewQuickReplyItems(
        linebot.NewQuickReplyButton(
            "",
            linebot.NewMessageAction("確認", "confirm_action"),
        ),
        linebot.NewQuickReplyButton(
            "",
            linebot.NewMessageAction("取消", "cancel_action"),
        ),
    )

    _, err := bot.PushMessage(userID, linebot.NewTextMessage(message).
        WithQuickReplies(quickReplyItems)).Do()
    return err
}
```

**參考資料**:
- [LINE Messaging API 文件](https://developers.line.biz/en/docs/messaging-api/)
- [line-bot-sdk-go GitHub](https://github.com/line/line-bot-sdk-go)

---

## 10. 測試策略

### 決策：多層次測試（Unit + Integration + Contract + E2E）

**測試框架選擇**:

1. **Unit Tests**: Go testing package + [testify](https://github.com/stretchr/testify)
   - 理由：標準庫輕量、testify 提供豐富的 assertion

2. **Integration Tests**: [Testcontainers](https://golang.testcontainers.org/)
   - 理由：自動啟動 PostgreSQL Docker 容器，真實資料庫測試

3. **Contract Tests**: [httptest](https://pkg.go.dev/net/http/httptest)
   - 理由：Mock HTTP 請求/回應，驗證 API 契約

4. **E2E Tests**: 手動測試 checklist
   - 理由：Zoom 與 ffmpeg 需實際硬體環境，自動化成本過高

**測試範例** (Unit Test):
```go
func TestValidateSignature(t *testing.T) {
    secret := "my-secret"
    body := []byte(`{"events":[]}`)

    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(body)
    validSig := base64.StdEncoding.EncodeToString(mac.Sum(nil))

    assert.True(t, ValidateSignature(body, validSig, secret))
    assert.False(t, ValidateSignature(body, "invalid-sig", secret))
}
```

**測試範例** (Integration Test with Testcontainers):
```go
func TestMeetingRecordRepository(t *testing.T) {
    ctx := context.Background()

    // 啟動 PostgreSQL 容器
    postgresC, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: testcontainers.ContainerRequest{
            Image:        "postgres:15",
            ExposedPorts: []string{"5432/tcp"},
            Env: map[string]string{
                "POSTGRES_PASSWORD": "password",
            },
        },
        Started: true,
    })
    require.NoError(t, err)
    defer postgresC.Terminate(ctx)

    // 連接資料庫並測試
    // ...
}
```

**參考資料**:
- [Testcontainers for Go](https://golang.testcontainers.org/)
- [Testify GitHub](https://github.com/stretchr/testify)

---

## 總結

### 技術棧全貌

| 層級 | 技術選擇 | 主要理由 |
|------|----------|----------|
| **架構** | 雲地混合 (AWS + 本地 Mac) | 符合需求、成本效益 |
| **通訊** | Tailscale VPN | 零配置、企業級安全 |
| **語言** | Golang 1.21+ | 效能、部署簡單 |
| **框架** | Gin | 高效能、開發體驗佳 |
| **ORM** | XORM | 輕量、功能足夠 |
| **資料庫** | PostgreSQL 15+ | 企業級穩定性、JSONB 支援 |
| **Zoom 控制** | AppleScript | macOS 原生、穩定 |
| **錄製** | ffmpeg + BlackHole | 開源、功能完整 |
| **部署** | AWS EC2 + launchd | 成本可控、可靠性高 |
| **LINE 整合** | line-bot-sdk-go | 官方 SDK、文件完善 |
| **測試** | testify + Testcontainers | 多層次測試保證品質 |

### 未解決問題（無）

所有技術選擇已完成研究，無 "NEEDS CLARIFICATION" 項目。

### 下一步

進入 **Phase 1: Design & Contracts**，產出：
- `data-model.md`: PostgreSQL Schema 設計
- `contracts/gateway-api.yaml`: Gateway Service OpenAPI 規格
- `contracts/mac-api.yaml`: Mac Control Service OpenAPI 規格
- `quickstart.md`: 快速開始指南
