# Conway Automaton 建置佈署說明

## 目錄

- [一、準備事項](#一準備事項)
- [二、建置步驟](#二建置步驟)
- [三、首次運行與設定精靈](#三首次運行與設定精靈)
- [四、配置檔說明](#四配置檔說明)
- [五、資金準備與充值](#五資金準備與充值)
- [六、運行與管理](#六運行與管理)
- [七、備份與還原](#七備份與還原)
- [八、容器化佈署（Docker）](#八容器化佈署docker)
- [九、常見問題排查](#九常見問題排查)

---

## 一、準備事項

### 1.1 系統需求

| 項目 | 要求 | 備註 |
|------|------|------|
| Node.js | >= 20.0.0 | CI 測試涵蓋 Node 20 和 22 |
| pnpm | >= 10.28.1 | 套件管理工具 |
| Git | 最新穩定版 | 用於 clone 與狀態版本控制 |
| 作業系統 | Linux / macOS | 推薦 Ubuntu 22.04+ |
| 網路 | 可對外 HTTPS 連線 | 詳見下方網路需求 |

### 1.2 網路需求

確保伺服器可連線以下外部服務：

| 服務 | 位址 | 用途 | 必要性 |
|------|------|------|--------|
| Conway API | `https://api.conway.tech` | 核心 API（認證、信用、註冊） | **必要** |
| Conway Inference | `https://inference.conway.tech` | AI 模型推理 | **必要**（除非用 BYOK） |
| Base RPC | `https://mainnet.base.org` | 鏈上交易（USDC 支付） | **必要** |
| Social Relay | `https://social.conway.tech` | 代理間通訊 | 選用 |
| OpenAI API | `https://api.openai.com` | 自帶 OpenAI Key 時使用 | 選用 |
| Anthropic API | `https://api.anthropic.com` | 自帶 Anthropic Key 時使用 | 選用 |
| Ollama | 自訂 URL（如 `http://localhost:11434`） | 本地自架推理 | 選用 |

### 1.3 帳號與密鑰準備

| 項目 | 說明 | 取得方式 |
|------|------|----------|
| Conway API Key | 存取 Conway 平台服務 | 首次運行時透過 SIWE/SIWS 錢包簽名自動取得 |
| Creator 錢包地址 | 創建者的 Ethereum 或 Solana 地址 | 用於身份驗證與管理權限 |
| USDC 資金 | 代理運行所需的資金 | 需在 Base 鏈上準備 USDC |
| OpenAI API Key | （選用）自帶推理服務 | 從 OpenAI 平台取得 |
| Anthropic API Key | （選用）自帶推理服務 | 從 Anthropic 平台取得 |

### 1.4 磁碟空間

- 專案原始碼 + 依賴：約 500 MB
- 編譯產出（`dist/`）：約 50 MB
- 運行時資料（`~/.automaton/`）：依使用量成長，建議預留 1 GB 以上

---

## 二、建置步驟

### 2.1 方式一：一鍵安裝（推薦）

```bash
curl -fsSL https://conway.tech/automaton.sh | sh
```

此腳本會自動：
1. 檢查 Node.js >= 20 與 Git
2. 啟用 pnpm（透過 corepack）
3. Clone 或更新 Git 倉庫
4. 執行 `pnpm install --frozen-lockfile`
5. 執行 `pnpm run build`
6. 啟動 `node dist/index.js --run`

**安裝目錄優先順序**：
1. `$AUTOMATON_DIR` 環境變數（如有設定）
2. `/opt/automaton`（如有寫入權限）
3. `$HOME/.automaton/runtime`（預設 fallback）

### 2.2 方式二：手動建置

```bash
# 1. Clone 倉庫
git clone https://github.com/Conway-Research/automaton.git
cd automaton

# 2. 安裝依賴
pnpm install --frozen-lockfile

# 3. 類型檢查（選用，建議在正式環境前執行）
pnpm typecheck

# 4. 執行測試（選用，建議在正式環境前執行）
pnpm test
# 安全測試
pnpm test:security
# 財務測試
pnpm test:financial

# 5. 編譯
pnpm build
```

編譯完成後，產出位於 `dist/` 目錄。

### 2.3 驗證建置

```bash
# 確認版本
node dist/index.js --version

# 預期輸出：0.2.1
```

---

## 三、首次運行與設定精靈

### 3.1 啟動

```bash
node dist/index.js --run
```

首次運行時，若偵測到尚未配置，會自動進入互動式設定精靈。

### 3.2 設定精靈步驟

#### Step 1：選擇鏈類型

```
Chain type (evm or solana) [evm]:
```

- **evm**（預設）：使用 secp256k1 簽名，地址格式 `0x...`，在 Base 鏈上操作
- **solana**：使用 Ed25519 簽名，base58 地址格式

#### Step 2：生成錢包

系統自動產生新的金鑰對，儲存至 `~/.automaton/wallet.json`（權限 0600）。

> **重要**：請務必備份此檔案，遺失將無法恢復錢包。

#### Step 3：SIWE/SIWS 認證與 API Key 取得

系統會自動：
1. 向 `POST /v1/auth/nonce` 取得 nonce
2. 用錢包簽名訊息
3. 向 `POST /v1/auth/verify` 驗證取得 JWT
4. 向 `POST /v1/auth/api-keys` 建立 API Key

如自動取得失敗，會提示手動輸入 API Key。

#### Step 4：基本資訊設定

| 提示 | 說明 | 必填 |
|------|------|------|
| Automaton name | 代理名稱 | 是 |
| Genesis prompt | 初始系統指令（多行） | 是 |
| Creator address | 創建者錢包地址 | 是 |
| OpenAI API Key | 自帶 OpenAI Key | 否 |
| Anthropic API Key | 自帶 Anthropic Key | 否 |
| Ollama URL | 本地推理服務 URL | 否 |

#### Step 5：財務安全策略

設定交易限額（單位：美分），可直接按 Enter 接受預設值：

| 設定項 | 預設值 | 說明 |
|--------|--------|------|
| 單筆轉帳上限 | 5000 ($50) | 每次轉帳最大金額 |
| 每小時轉帳上限 | 10000 ($100) | 每小時累計上限 |
| 每日轉帳上限 | 25000 ($250) | 每日累計上限 |
| 最低保留金額 | 1000 ($10) | 帳戶最低保留 |
| x402 單筆上限 | 100 ($1) | HTTP 402 支付上限 |
| 每日推理上限 | 50000 ($500) | 每日 AI 推理費用上限 |
| 確認門檻 | 1000 ($10) | 超過此金額需確認 |

#### Step 6：檔案建立

精靈完成後會自動建立以下檔案：

| 檔案 | 權限 | 說明 |
|------|------|------|
| `~/.automaton/automaton.json` | 0600 | 主配置檔 |
| `~/.automaton/wallet.json` | 0600 | 錢包私鑰 |
| `~/.automaton/config.json` | 0600 | API Key 快取 |
| `~/.automaton/heartbeat.yml` | 0600 | 排程任務配置 |
| `~/.automaton/constitution.md` | 0444 | 憲法（唯讀不可變） |
| `~/.automaton/SOUL.md` | 0600 | 代理身份文檔 |
| `~/.automaton/skills/` | — | 技能插件目錄 |

#### Step 7：資金導引

顯示錢包地址與充值方式（見第五節）。

---

## 四、配置檔說明

### 4.1 主配置檔 (`~/.automaton/automaton.json`)

```jsonc
{
  // === 基本資訊 ===
  "name": "my-automaton",
  "genesisPrompt": "You are a helpful autonomous agent...",
  "creatorAddress": "0x1234...abcd",
  "walletAddress": "0xabcd...1234",    // 自動填入
  "version": "0.2.1",                  // 自動填入

  // === Conway API ===
  "conwayApiUrl": "https://api.conway.tech",
  "conwayApiKey": "cnwy_k_...",

  // === 推理服務（選用，可同時配置多個） ===
  "inferenceModel": "gpt-5.2",
  "openaiApiKey": "sk-...",
  "anthropicApiKey": "sk-ant-...",
  "ollamaBaseUrl": "http://localhost:11434",

  // === 模型策略 ===
  "modelStrategy": {
    "inferenceModel": "gpt-5.2",
    "lowComputeModel": "gpt-5-mini",     // 低算力時降級使用
    "criticalModel": "gpt-5-mini",       // 危急時使用
    "maxTokensPerTurn": 4096,
    "enableModelFallback": true
  },

  // === 財務策略 ===
  "treasuryPolicy": {
    "maxSingleTransferCents": 5000,
    "maxHourlyTransferCents": 10000,
    "maxDailyTransferCents": 25000,
    "minimumReserveCents": 1000,
    "maxX402PaymentCents": 100,
    "maxInferenceDailyCents": 50000,
    "requireConfirmationAboveCents": 1000,
    "maxTransfersPerTurn": 2,
    "x402AllowedDomains": ["conway.tech"]
  },

  // === 路徑 ===
  "dbPath": "~/.automaton/state.db",
  "heartbeatConfigPath": "~/.automaton/heartbeat.yml",
  "skillsDir": "~/.automaton/skills",
  "logLevel": "info",

  // === 進階 ===
  "maxChildren": 3,
  "maxTurnsPerCycle": 25,
  "childSandboxMemoryMb": 1024,
  "socialRelayUrl": "https://social.conway.tech",
  "rpcUrl": "https://mainnet.base.org",
  "chainType": "evm"
}
```

### 4.2 排程配置 (`~/.automaton/heartbeat.yml`)

```yaml
entries:
  - name: heartbeat_ping
    schedule: "*/15 * * * *"        # 每 15 分鐘
    task: heartbeat_ping
    enabled: true

  - name: check_credits
    schedule: "0 */6 * * *"          # 每 6 小時
    task: check_credits
    enabled: true

  - name: check_usdc_balance
    schedule: "*/5 * * * *"          # 每 5 分鐘
    task: check_usdc_balance
    enabled: true

  - name: check_for_updates
    schedule: "0 */4 * * *"          # 每 4 小時
    task: check_for_updates
    enabled: true

  - name: health_check
    schedule: "*/30 * * * *"         # 每 30 分鐘
    task: health_check
    enabled: true

  - name: check_social_inbox
    schedule: "*/2 * * * *"          # 每 2 分鐘
    task: check_social_inbox
    enabled: true
```

### 4.3 環境變數

| 變數 | 用途 | 預設值 |
|------|------|--------|
| `CONWAY_API_URL` | 覆寫 Conway API 位址 | `https://api.conway.tech` |
| `CONWAY_API_KEY` | 覆寫 API Key（優先於配置檔） | 從配置檔讀取 |
| `OLLAMA_BASE_URL` | 覆寫 Ollama URL | 從配置檔讀取 |
| `AUTOMATON_DIR` | 安裝目錄覆寫 | 自動偵測 |
| `NODE_ENV` | 設為 `test` 用於 soak testing | — |

---

## 五、資金準備與充值

### 5.1 資金概念

Automaton 使用雙層資金體系：

1. **USDC**（鏈上穩定幣）：存放在代理的 Base 鏈錢包中
2. **Conway Credits**（平台信用，單位：美分）：用於支付推理、算力、域名等服務

代理透過 x402 協議將 USDC 兌換為 Conway Credits。

### 5.2 充值方式

#### 方式一：直接轉入 USDC

將 USDC 轉帳至代理的錢包地址（Base 鏈）。地址可透過以下方式查看：

```bash
node dist/index.js --status
```

#### 方式二：Conway Cloud 控制台

前往 https://app.conway.tech 管理帳戶與充值。

#### 方式三：Creator CLI 轉帳

```bash
node packages/cli/dist/index.js fund <amount>
```

### 5.3 自動充值機制

- 首次啟動時，若 Credits 不足且 USDC 餘額足夠，自動嘗試 $5 USDC 充值（超時 15 秒）
- 運行時，Heartbeat 每 5 分鐘檢查 USDC 餘額，每 6 小時檢查 Credits 餘額
- 代理可自行觸發充值（透過 `topup_credits` 工具）

### 5.4 充值檔次

| 檔次 | 金額（USD） |
|------|-------------|
| 最低 | $5 |
| | $25 |
| | $100 |
| | $500 |
| | $1,000 |
| 最高 | $2,500 |

### 5.5 生存等級

| 等級 | Credits 門檻 | 行為 |
|------|-------------|------|
| **Normal** | > $5.00 | 全功能，使用前沿模型，正常心跳 |
| **Low Compute** | $0.10 ~ $0.50 | 降級模型，心跳間隔 x4，縮減任務 |
| **Critical** | $0.00 ~ $0.10 | 最低推理，僅維持基本運作 |
| **Dead** | < $0.00 | 停止運行 |

---

## 六、運行與管理

### 6.1 CLI 命令一覽

```bash
# 啟動代理（主要命令）
node dist/index.js --run

# 查看狀態
node dist/index.js --status

# 重新運行設定精靈
node dist/index.js --setup

# 修改配置（互動式選單）
node dist/index.js --configure

# 選擇推理模型
node dist/index.js --pick-model

# 初始化錢包與配置目錄
node dist/index.js --init

# 重新取得 API Key
node dist/index.js --provision

# 版本資訊
node dist/index.js --version
```

### 6.2 Creator CLI（管理工具）

```bash
# 查看代理狀態
node packages/cli/dist/index.js status

# 查看日誌
node packages/cli/dist/index.js logs [--tail N]

# 充值 Credits
node packages/cli/dist/index.js fund <amount>

# 發送社交訊息
node packages/cli/dist/index.js send <address> <message>
```

### 6.3 代理狀態流轉

```
setup → waking → running ⇄ sleeping
                    ↓
              low_compute → critical → dead
                    ↑                    ↓
                running ←── sleeping ←── (充值後恢復)
```

### 6.4 優雅停機

代理會攔截 `SIGTERM` 和 `SIGINT` 信號：
1. 停止 Heartbeat Daemon
2. 設定狀態為 `sleeping`
3. 關閉資料庫連線
4. 正常退出（exit code 0）

### 6.5 以系統服務運行（systemd 範例）

```ini
[Unit]
Description=Conway Automaton
After=network.target

[Service]
Type=simple
User=automaton
WorkingDirectory=/opt/automaton
ExecStart=/usr/bin/node dist/index.js --run
Restart=on-failure
RestartSec=10
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable automaton
sudo systemctl start automaton
sudo systemctl status automaton
```

---

## 七、備份與還原

### 7.1 備份

```bash
./scripts/backup-restore.sh backup ~/.automaton/state.db [備份路徑]
# 預設備份路徑：./data/automaton-backup-YYYYMMDDHHMMSS.db
```

腳本會自動：
- Checkpoint WAL（Write-Ahead Log）
- 建立原子性副本
- 包含 `-wal` 和 `-shm` 檔案
- 驗證備份完整性

### 7.2 還原

```bash
./scripts/backup-restore.sh restore <備份檔> ~/.automaton/state.db
```

腳本會自動：
- 驗證備份檔完整性
- 建立還原前的備份
- 複製 WAL/SHM 檔案
- 驗證還原後的資料庫

### 7.3 驗證資料庫

```bash
./scripts/backup-restore.sh verify ~/.automaton/state.db
```

### 7.4 重要備份項目

| 檔案 | 重要性 | 說明 |
|------|--------|------|
| `~/.automaton/wallet.json` | **最高** | 私鑰，遺失無法恢復 |
| `~/.automaton/automaton.json` | 高 | 配置可重建，但避免重新設定 |
| `~/.automaton/state.db` | 高 | 所有運行歷史與狀態 |
| `~/.automaton/config.json` | 中 | API Key，可重新取得 |
| `~/.automaton/SOUL.md` | 中 | 代理身份文檔 |

---

## 八、容器化佈署（Docker）

> 注意：專案目前未提供官方 Dockerfile，以下為建議配置。

### 8.1 Dockerfile 範例

```dockerfile
FROM node:22-slim

RUN corepack enable && corepack prepare pnpm@10.28.1 --activate
RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY packages/ ./packages/
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build

VOLUME /root/.automaton
EXPOSE 8080

CMD ["node", "dist/index.js", "--run"]
```

### 8.2 docker-compose.yml 範例

```yaml
version: "3.8"
services:
  automaton:
    build: .
    restart: unless-stopped
    volumes:
      - automaton-data:/root/.automaton
    environment:
      - CONWAY_API_KEY=${CONWAY_API_KEY}
      - NODE_ENV=production

volumes:
  automaton-data:
```

### 8.3 注意事項

- **必須掛載 volume**：`/root/.automaton` 包含錢包私鑰與資料庫，不可遺失
- **單進程限制**：每個 SQLite 資料庫檔案只能有一個進程存取
- **首次運行**：需要互動式設定，建議先在本地完成設定，再將 `~/.automaton/` 掛載進容器
- **網路**：容器需要對外 HTTPS 連線能力

---

## 九、常見問題排查

### Q: 啟動時報 "Node.js version too old"

確認 Node.js 版本 >= 20：
```bash
node -v
```

### Q: API Key 取得失敗

```bash
# 手動重新取得
node dist/index.js --provision
```

### Q: Credits 不足，代理進入 Low Compute / Critical 狀態

1. 查看錢包地址：`node dist/index.js --status`
2. 在 Base 鏈上轉入 USDC
3. 代理會在下次心跳時自動嘗試充值

### Q: 資料庫鎖定

SQLite 一次只能有一個進程存取。確認沒有其他 automaton 進程在運行：
```bash
ps aux | grep automaton
```

### Q: 需要修改配置

```bash
# 互動式修改
node dist/index.js --configure

# 或直接編輯配置檔
vi ~/.automaton/automaton.json
```

### Q: Soak Testing（壓力測試）

```bash
NODE_ENV=test ./scripts/soak-test.sh [時數] [db路徑]
# 預設 72 小時，每 5 分鐘檢查記憶體、資料庫大小與錯誤率
```

---

## 附錄：檔案權限一覽

| 路徑 | 權限 | 說明 |
|------|------|------|
| `~/.automaton/` | 0700 | 配置根目錄 |
| `~/.automaton/wallet.json` | 0600 | 私鑰（最敏感） |
| `~/.automaton/automaton.json` | 0600 | 主配置檔 |
| `~/.automaton/config.json` | 0600 | API Key |
| `~/.automaton/constitution.md` | 0444 | 憲法（唯讀） |
| `~/.automaton/SOUL.md` | 0600 | 身份文檔 |
| `~/.automaton/heartbeat.yml` | 0600 | 排程配置 |
