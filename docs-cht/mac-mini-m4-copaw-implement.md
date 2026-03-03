# Mac Mini M4 CoPaw 完整實作指南

> 本指南詳細說明如何在 **Mac Mini M4 (16GB)** 上從無到有部署 CoPaw，整合 Ollama 本地模型、**遠端 Ollama（另一台 Mac Mini M4 24GB）**、Claude API、Discord 機器人、Gmail 自動收發、GitHub 專案上傳與通知。

---

## 目錄

- [前置環境準備](#前置環境準備)
- [第一部分：安裝 CoPaw](#第一部分安裝-copaw)
- [第二部分：設定 AI 引擎](#第二部分設定-ai-引擎)
  - [方案 A：Ollama 本地模型](#方案-aollama-本地模型)
  - [方案 B：Claude API（Anthropic）](#方案-bclaude-apianthropic)
  - [方案 C：遠端 Ollama（24GB 機器）](#方案-c遠端-ollama24gb-機器)
- [第三部分：設定 Discord 機器人](#第三部分設定-discord-機器人)
- [第四部分：設定 Gmail 自動收發](#第四部分設定-gmail-自動收發)
- [第五部分：設定 GitHub 專案自動上傳](#第五部分設定-github-專案自動上傳)
- [第六部分：設定自動化工作流程](#第六部分設定自動化工作流程)
- [第七部分：啟動與驗證](#第七部分啟動與驗證)
- [第八部分：進階設定與維運](#第八部分進階設定與維運)
- [常見問題排除](#常見問題排除)
- [完整設定檔參考](#完整設定檔參考)

---

## 前置環境準備

### 硬體確認

Mac Mini M4 16GB 適合執行 7B 至 14B 參數的本地模型。若使用 Claude API 作為主力引擎，16GB 完全足夠。若另有 **Mac Mini M4 24GB**，可將 Ollama 部署於其上，本機 CoPaw 指向遠端，使用 14B ~ 32B 等更大模型（見[方案 C](#方案-c遠端-ollama24gb-機器)）。

### 安裝基礎工具

```bash
# 安裝 Homebrew（如尚未安裝）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 安裝 Python 3.12（CoPaw 需要 3.10 ~ 3.13）
brew install python@3.12

# 安裝 Git（後續 GitHub 操作需要）
brew install git

# 安裝 GitHub CLI（用於 GitHub 操作）
brew install gh

# 安裝 Node.js（前端開發或 MCP 工具需要）
brew install node
```

---

## 第一部分：安裝 CoPaw

### 方式一：一鍵安裝（推薦，免設定 Python 環境）

```bash
curl -fsSL https://copaw.agentscope.io/install.sh | bash
```

### 方式二：pip 安裝（開發者推薦）

```bash
# 建立虛擬環境
python3.12 -m venv ~/copaw-env
source ~/copaw-env/bin/activate

# 安裝 CoPaw（含 Ollama 支援）
pip install "copaw[ollama]"

# 安裝 Playwright 瀏覽器（browser_use 工具需要）
playwright install chromium
```

### 初始化工作目錄

```bash
copaw init
```

互動式初始化會引導你完成以下設定：

1. **安全聲明確認**：CoPaw 的 Agent 可以執行 Shell 指令，確認你了解風險
2. **選擇語言**：選 `zh`（中文）
3. **設定 LLM Provider**：此步驟先跳過，後面會詳細設定
4. **選擇 Skills**：建議選 `all`（啟用所有內建 Skills）

初始化完成後，工作目錄在 `~/.copaw/`。

> **快速初始化**（使用預設值）：`copaw init --defaults`

---

## 第二部分：設定 AI 引擎

CoPaw 提供多種 AI 引擎方案，你可以擇一使用，也可以並存、隨時切換。若你另有 **Mac Mini M4 24GB** 作為高效能機器，可將 Ollama 部署於其上，讓 16GB 本機的 CoPaw 指向遠端，使用 14B ~ 32B 等更大模型。

### 方案 A：Ollama 本地模型

Ollama 可讓你在 Mac Mini M4 上離線運行開源 LLM，資料不外傳。

#### 步驟 A1：安裝 Ollama

```bash
brew install ollama
```

#### 步驟 A2：啟動 Ollama 服務

```bash
# 以背景服務方式啟動（推薦）
brew services start ollama

# 或手動啟動（開發除錯用）
ollama serve
```

確認服務運行：

```bash
curl http://localhost:11434/api/tags
# 應回傳 JSON，表示服務正常
```

#### 步驟 A3：下載適合 M4 16GB 的模型

Mac Mini M4 16GB 建議使用 7B ~ 14B 參數的模型。以下是推薦選擇：

```bash
# 推薦：Qwen 3（支援中文，能力均衡）
ollama pull qwen3:8b

# 推薦：Qwen 2.5 Coder（程式撰寫能力強）
ollama pull qwen2.5-coder:7b

# 可選：Llama 3.1（英文能力優秀）
ollama pull llama3.1:8b

# 可選：Deepseek-coder-v2（程式與推理）
ollama pull deepseek-coder-v2:16b
# ⚠ 16B 會使用約 10GB 記憶體，酌情選用
```

也可以透過 CoPaw CLI 下載：

```bash
copaw models ollama-pull qwen3:8b
```

確認模型已下載：

```bash
ollama list
# 或
copaw models ollama-list
```

#### 步驟 A4：設定 Ollama 為 CoPaw 的 AI 引擎

**方式一：CLI 互動設定（推薦）**

```bash
copaw models set-llm
# 選擇 provider: ollama
# 選擇 model: qwen3:8b（或你下載的模型）
```

**方式二：直接編輯 `~/.copaw/providers.json`**

```json
{
  "providers": {
    "ollama": {
      "base_url": "http://localhost:11434/v1",
      "api_key": "",
      "extra_models": []
    }
  },
  "custom_providers": {},
  "active_llm": {
    "provider_id": "ollama",
    "model": "qwen3:8b"
  }
}
```

> **注意**：Ollama 的 `base_url` 預設為 `http://localhost:11434/v1`，不需要 API Key。CoPaw 會自動同步 Ollama 已下載的模型清單。

**方式三：透過 API**

啟動 CoPaw 後：

```bash
curl -X PUT http://127.0.0.1:8088/api/models/active \
  -H "Content-Type: application/json" \
  -d '{"active_llm": {"provider_id": "ollama", "model": "qwen3:8b"}}'
```

#### 步驟 A5：Ollama 效能調校（M4 專屬）

M4 晶片原生支援 Metal GPU 加速，Ollama 預設已啟用。若需微調：

```bash
# 設定最大並行請求數（預設 1，不建議在 16GB 上增加）
export OLLAMA_NUM_PARALLEL=1

# 設定模型在記憶體中的保留時間（預設 5m）
export OLLAMA_KEEP_ALIVE=30m

# 將以上加入 ~/.zshrc 使其永久生效
echo 'export OLLAMA_NUM_PARALLEL=1' >> ~/.zshrc
echo 'export OLLAMA_KEEP_ALIVE=30m' >> ~/.zshrc
```

---

### 方案 B：Claude API（Anthropic）

CoPaw **沒有內建 Anthropic Provider**，需要建立自訂 Provider。Claude 的優勢在於推理能力強、中文品質高、不佔本地資源。

#### 步驟 B1：取得 Claude API Key

1. 前往 [Anthropic Console](https://console.anthropic.com/)
2. 註冊/登入帳號
3. 進入 **API Keys** 頁面
4. 點擊 **Create Key**，複製產生的 API Key（格式：`sk-ant-...`）

> **Claude 訂閱方案**：使用 API 需要 Anthropic 的付費方案（非 Claude Pro/Team 訂閱）。API 用量依使用量計費。

#### 步驟 B2：在 CoPaw 建立 Anthropic 自訂 Provider

**方式一：CLI 互動設定**

```bash
copaw models add-provider anthropic \
  -n "Anthropic (Claude)" \
  -u "https://api.anthropic.com/v1" \
  --api-key-prefix "sk-ant-"
```

然後設定 API Key：

```bash
copaw models config-key anthropic
# 輸入你的 API Key：sk-ant-xxxxx
```

新增 Claude 模型：

```bash
# 新增 Claude Sonnet 4（推薦，性價比高）
copaw models add-model anthropic -m "claude-sonnet-4-20250514" -n "Claude Sonnet 4"

# 新增 Claude Opus 4（最強能力）
copaw models add-model anthropic -m "claude-opus-4-20250514" -n "Claude Opus 4"

# 新增 Claude Haiku 3.5（快速便宜）
copaw models add-model anthropic -m "claude-3-5-haiku-20241022" -n "Claude 3.5 Haiku"
```

設為目前使用的模型：

```bash
copaw models set-llm
# 選擇 provider: anthropic
# 選擇 model: claude-sonnet-4-20250514
```

**方式二：透過 API 建立**

啟動 CoPaw 後：

```bash
# 建立自訂 Provider
curl -X POST http://127.0.0.1:8088/api/models/custom-providers \
  -H "Content-Type: application/json" \
  -d '{
    "id": "anthropic",
    "name": "Anthropic (Claude)",
    "default_base_url": "https://api.anthropic.com/v1",
    "api_key_prefix": "sk-ant-",
    "models": [
      {"id": "claude-sonnet-4-20250514", "name": "Claude Sonnet 4"},
      {"id": "claude-opus-4-20250514", "name": "Claude Opus 4"},
      {"id": "claude-3-5-haiku-20241022", "name": "Claude 3.5 Haiku"}
    ],
    "base_url": "https://api.anthropic.com/v1",
    "api_key": "sk-ant-你的API Key"
  }'

# 設為啟用模型
curl -X PUT http://127.0.0.1:8088/api/models/active \
  -H "Content-Type: application/json" \
  -d '{"active_llm": {"provider_id": "anthropic", "model": "claude-sonnet-4-20250514"}}'
```

**方式三：直接編輯 `~/.copaw/providers.json`**

```json
{
  "providers": {
    "ollama": {
      "base_url": "http://localhost:11434/v1",
      "api_key": "",
      "extra_models": []
    }
  },
  "custom_providers": {
    "anthropic": {
      "id": "anthropic",
      "name": "Anthropic (Claude)",
      "default_base_url": "https://api.anthropic.com/v1",
      "api_key_prefix": "sk-ant-",
      "models": [
        { "id": "claude-sonnet-4-20250514", "name": "Claude Sonnet 4" },
        { "id": "claude-opus-4-20250514", "name": "Claude Opus 4" },
        { "id": "claude-3-5-haiku-20241022", "name": "Claude 3.5 Haiku" }
      ],
      "base_url": "https://api.anthropic.com/v1",
      "api_key": "sk-ant-你的API Key",
      "chat_model": "OpenAIChatModel"
    }
  },
  "active_llm": {
    "provider_id": "anthropic",
    "model": "claude-sonnet-4-20250514"
  }
}
```

> **注意**：CoPaw 使用 OpenAI 相容介面（`OpenAIChatModel`）呼叫 Claude API。Anthropic 的 `/v1` 端點支援 OpenAI 相容模式。若遇到問題，可嘗試使用第三方 OpenAI 相容 proxy（如 LiteLLM）。

#### 步驟 B3：驗證 Claude 連線

```bash
# 測試 Provider 連線（需先啟動 CoPaw）
copaw app &
curl -X POST http://127.0.0.1:8088/api/models/anthropic/test
```

---

### 方案 C：遠端 Ollama（24GB 機器）

若你擁有另一台 **Mac Mini M4 24GB**，可將 Ollama 部署於其上，讓 16GB 本機的 CoPaw 透過網路呼叫遠端模型，使用 14B ~ 32B 等更大參數的模型，而不佔用本機記憶體。

#### 架構說明

```
┌─────────────────────────────────┐    網路 (LAN)    ┌─────────────────────────────────┐
│  Mac Mini M4 16GB（本機）        │ ◄─────────────► │  Mac Mini M4 24GB（遠端）        │
│                                 │   HTTP :11434   │                                 │
│  • CoPaw 服務                    │                 │  • Ollama 服務                    │
│  • Discord / Gmail / GitHub      │                 │  • Qwen3 14B / 32B 等大模型       │
│  • 本地 Ollama（可選，7B~8B）    │                 │  • 本地 Embedding（可選）         │
└─────────────────────────────────┘                 └─────────────────────────────────┘
```

**優勢**：16GB 本機不跑大模型，記憶體留給 CoPaw、瀏覽器與其他應用；24GB 遠端專心跑 14B ~ 32B 模型，效能與品質兼顧。

---

#### 遠端端（24GB 機器）設定

在 **Mac Mini M4 24GB** 上完成以下設定，使其成為 Ollama 伺服器。

##### 步驟 C1：安裝並啟動 Ollama

```bash
# 在 24GB 機器上執行
brew install ollama
brew services start ollama
```

##### 步驟 C2：設定 Ollama 接受遠端連線

Ollama 預設只監聽 `127.0.0.1`，需設定 `OLLAMA_HOST` 才能接受來自其他機器的連線。

**方式一：編輯 Homebrew 的 plist（推薦）**

```bash
# 在 24GB 機器上執行
# plist 路徑：Apple Silicon 為 /opt/homebrew/opt/ollama/，Intel 為 /usr/local/opt/ollama/
sudo nano /opt/homebrew/opt/ollama/homebrew.mxcl.ollama.plist
```

在 `<dict>` 最外層內加入（若已有 `EnvironmentVariables`，則在該 dict 內加入 `OLLAMA_HOST`）：

```xml
<key>EnvironmentVariables</key>
<dict>
    <key>OLLAMA_HOST</key>
    <string>0.0.0.0:11434</string>
</dict>
```

儲存後重啟服務：

```bash
brew services restart ollama
```

> **注意**：若使用 `brew upgrade ollama` 更新，Homebrew 的 plist 可能被覆蓋，需重新加入 `OLLAMA_HOST`。若需長期穩定，建議改用下方自訂 LaunchAgent。

**方式二：自訂 LaunchAgent（更易維護）**

若不想修改 Homebrew 的 plist，可建立自訂 plist：

```bash
# 停止 Homebrew 管理的 Ollama
brew services stop ollama

# 建立自訂 LaunchAgent
cat > ~/Library/LaunchAgents/com.ollama.remote.plist << 'PLIST'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.ollama.remote</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/ollama</string>
        <string>serve</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/ollama.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/ollama-error.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>OLLAMA_HOST</key>
        <string>0.0.0.0:11434</string>
        <key>OLLAMA_NUM_PARALLEL</key>
        <string>2</string>
        <key>OLLAMA_KEEP_ALIVE</key>
        <string>1h</string>
    </dict>
</dict>
</plist>
PLIST

launchctl load ~/Library/LaunchAgents/com.ollama.remote.plist
```

> **注意**：Intel Mac 的 ollama 路徑可能為 `/usr/local/bin/ollama`，請依實際安裝路徑調整。

##### 步驟 C3：防火牆放行 11434 埠

```bash
# 在 24GB 機器上執行
# 若使用 macOS 內建防火牆，需允許傳入連線
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /opt/homebrew/bin/ollama
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --unblockapp /opt/homebrew/bin/ollama

# 或於「系統設定 → 網路 → 防火牆 → 選項」中，允許 ollama 接受傳入連線
```

##### 步驟 C4：下載 24GB 適用模型

在 24GB 機器上執行（詳見 [24GB 實作指南](./mac-mini-m4-24g-copaw-implement.md#第三部分24gb-模型完整選擇指南)）：

```bash
# 在 24GB 機器上執行
ollama pull qwen3:14b           # 主力對話（~8.5GB）
ollama pull qwen2.5-coder:14b   # 程式開發（~8.5GB）
ollama pull qwen3:32b           # 極致品質（~18GB，可選）
ollama pull nomic-embed-text    # 本地 Embedding（可選）
```

##### 步驟 C5：取得遠端 IP 並驗證

```bash
# 在 24GB 機器上執行，取得 IP
ifconfig | grep "inet " | grep -v 127.0.0.1
# 例如：192.168.1.100

# 在本機（16GB）上測試遠端連線
curl http://192.168.1.100:11434/api/tags
# 應回傳 JSON，列出遠端已下載的模型
```

---

#### 本地端（16GB 機器）設定

在 **Mac Mini M4 16GB**（執行 CoPaw 的機器）上完成以下設定。

##### 方式一：新增「遠端 Ollama」自訂 Provider（推薦）

保留本地 Ollama，同時新增遠端 Provider，可隨時切換：

```bash
# 在 16GB 機器上執行
# 1. 新增自訂 Provider（將 192.168.1.100 替換為你的 24GB 機器 IP）
copaw models add-provider ollama-remote \
  -n "Ollama 遠端 (24GB)" \
  -u "http://192.168.1.100:11434/v1" \
  --api-key-prefix ""

# 2. 新增遠端模型（需與 24GB 機器上 ollama list 的模型一致）
copaw models add-model ollama-remote -m "qwen3:14b" -n "Qwen3 14B"
copaw models add-model ollama-remote -m "qwen2.5-coder:14b" -n "Qwen 2.5 Coder 14B"
copaw models add-model ollama-remote -m "qwen3:32b" -n "Qwen3 32B"
# 依遠端實際下載的模型新增

# 3. 設為目前使用的模型
copaw models set-llm
# 選擇 provider: ollama-remote
# 選擇 model: qwen3:14b（或遠端有的模型）
```

**手動編輯 `~/.copaw/providers.json`**（若 CLI 不支援自訂 Provider 的 base_url，可直接編輯）：

```json
{
  "providers": {
    "ollama": {
      "base_url": "http://localhost:11434/v1",
      "api_key": "",
      "extra_models": []
    }
  },
  "custom_providers": {
    "ollama-remote": {
      "id": "ollama-remote",
      "name": "Ollama 遠端 (24GB)",
      "default_base_url": "http://192.168.1.100:11434/v1",
      "api_key_prefix": "",
      "models": [
        { "id": "qwen3:14b", "name": "Qwen3 14B" },
        { "id": "qwen2.5-coder:14b", "name": "Qwen 2.5 Coder 14B" },
        { "id": "qwen3:32b", "name": "Qwen3 32B" }
      ],
      "base_url": "http://192.168.1.100:11434/v1",
      "api_key": "",
      "chat_model": "OpenAIChatModel"
    }
  },
  "active_llm": {
    "provider_id": "ollama-remote",
    "model": "qwen3:14b"
  }
}
```

> **重要**：`models` 陣列需與遠端實際已下載的模型一致。在 24GB 機器上執行 `ollama list` 取得模型 ID。

##### 方式二：直接將內建 Ollama 指向遠端

若本機不跑 Ollama，可將內建 Provider 的 `base_url` 改為遠端：

```json
{
  "providers": {
    "ollama": {
      "base_url": "http://192.168.1.100:11434/v1",
      "api_key": "",
      "extra_models": []
    }
  },
  "active_llm": {
    "provider_id": "ollama",
    "model": "qwen3:14b"
  }
}
```

CoPaw 會自動從遠端同步模型清單。缺點是無法同時保留「本地 Ollama」與「遠端 Ollama」的切換。

---

#### 網路與安全考量

| 情境 | 建議 |
|------|------|
| **同一區域網路（LAN）** | 使用區網 IP（如 `192.168.1.100`），延遲低、設定簡單 |
| **不同網段 / VPN** | 確保 11434 埠可達，或透過 SSH 隧道（見下方） |
| **暴露於網際網路** | ⚠ 不建議。Ollama 無內建認證，若必須暴露，請使用反向代理 + 認證（如 nginx + basic auth） |
| **固定 IP** | 建議在路由器為 24GB 機器設定 DHCP 保留，避免 IP 變動 |

**SSH 隧道（跨網段 / 安全連線）**：

若 16GB 與 24GB 不在同一網段，可透過 SSH 隧道轉發：

```bash
# 在 16GB 機器上執行，建立隧道
ssh -L 11434:localhost:11434 user@24gb-machine-ip -N &

# 然後 CoPaw 的 base_url 設為 http://localhost:11434/v1
# 流量會經由 SSH 加密轉發到遠端
```

---

#### 遠端 Embedding（可選）

若 24GB 機器有運行 `nomic-embed-text`，可讓 16GB 本機的 CoPaw 使用遠端 Embedding：

```bash
# 在 16GB 機器上設定環境變數
copaw env set EMBEDDING_BASE_URL "http://192.168.1.100:11434/v1"
copaw env set EMBEDDING_API_KEY "ollama"
copaw env set EMBEDDING_MODEL_NAME "nomic-embed-text"
copaw env set EMBEDDING_DIMENSIONS "768"
copaw env set ENABLE_MEMORY_MANAGER "true"
```

---

### 切換 AI 引擎

設定好多個引擎後，可隨時透過前端或 CLI 切換：

```bash
copaw models set-llm

# 選 ollama → qwen3:8b           # 本地 Ollama（7B~8B）
# 選 ollama-remote → qwen3:14b   # 遠端 Ollama（14B~32B）
# 選 anthropic → claude-sonnet-4-20250514  # Claude API
```

---

## 第三部分：設定 Discord 機器人

### 步驟 1：建立 Discord Bot

1. 前往 [Discord Developer Portal](https://discord.com/developers/applications)
2. 點擊 **New Application**，命名（如 `CoPaw Assistant`）
3. 進入 **Bot** 頁面：
   - 點擊 **Reset Token**，複製產生的 Bot Token（**只會顯示一次**）
   - 開啟以下 **Privileged Gateway Intents**：
     - ✅ **Message Content Intent**
     - ✅ **Server Members Intent**（可選）
4. 進入 **OAuth2** → **URL Generator**：
   - Scopes 勾選：`bot`
   - Bot Permissions 勾選：
     - ✅ Send Messages
     - ✅ Read Message History
     - ✅ Attach Files
     - ✅ Embed Links
     - ✅ Read Messages/View Channels
   - 複製產生的 URL，在瀏覽器開啟並將 Bot 加入你的 Discord 伺服器

### 步驟 2：在 CoPaw 設定 Discord 頻道

**方式一：編輯 `~/.copaw/config.json`**

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "bot_token": "你的Discord Bot Token",
      "http_proxy": "",
      "http_proxy_auth": "",
      "bot_prefix": ""
    },
    "console": {
      "enabled": true
    }
  }
}
```

> **代理設定**：如果你的網路需要代理才能連接 Discord（例如在中國大陸），設定 `http_proxy`：
> ```json
> "http_proxy": "socks5://127.0.0.1:7890",
> "http_proxy_auth": ""
> ```

**方式二：透過 API 設定**

```bash
curl -X PUT http://127.0.0.1:8088/api/config/channels/discord \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": true,
    "bot_token": "你的Discord Bot Token",
    "http_proxy": "",
    "http_proxy_auth": "",
    "bot_prefix": ""
  }'
```

**方式三：CLI 互動設定**

```bash
copaw channels config
# 選擇 discord → 輸入 Bot Token
```

### 步驟 3：了解 Discord 頻道能力

CoPaw 的 Discord 頻道支援以下訊息類型：

| 功能 | 支援 | 說明 |
|------|------|------|
| 文字訊息 | ✅ | 接收/發送文字 |
| 圖片 | ✅ | 接收/發送圖片（png/jpg/gif/webp） |
| 影片 | ✅ | 接收影片（mp4/mov/mkv） |
| 音訊 | ✅ | 接收音訊（mp3/wav/m4a） |
| 檔案 | ✅ | 接收/發送任意檔案 |
| DM 私訊 | ✅ | 支援與 Bot 直接私訊 |
| 伺服器頻道 | ✅ | 支援在群組頻道中 @Bot 互動 |

### 步驟 4：與 Bot 互動測試

1. 啟動 CoPaw：`copaw app`
2. 在 Discord 中向 Bot 發送 DM（私訊）或在頻道中 @Bot
3. Bot 應回覆，確認連線正常

---

## 第四部分：設定 Gmail 自動收發

CoPaw 使用 **himalaya CLI** 作為郵件客戶端。我們需要安裝 himalaya、設定 Gmail 帳號，並建立支援發送功能的自訂 Skill。

### 步驟 1：安裝 himalaya CLI

```bash
brew install himalaya
```

確認安裝：

```bash
himalaya --version
```

### 步驟 2：取得 Gmail App Password

Gmail 需要使用「應用程式密碼」（App Password）而非一般密碼：

1. 前往 [Google 帳戶安全性](https://myaccount.google.com/security)
2. 確認已啟用**兩步驗證**
3. 前往 [應用程式密碼](https://myaccount.google.com/apppasswords)
4. 選擇「郵件」與「Mac」，點擊**產生**
5. 複製產生的 16 位密碼（格式：`xxxx xxxx xxxx xxxx`）

### 步驟 3：安全儲存密碼

使用 macOS 鑰匙圈儲存密碼（推薦）：

```bash
# 儲存 Gmail App Password 到 macOS 鑰匙圈
security add-generic-password \
  -a "你的Gmail@gmail.com" \
  -s "gmail-app-password" \
  -w "xxxx xxxx xxxx xxxx"
```

測試讀取：

```bash
security find-generic-password -a "你的Gmail@gmail.com" -s "gmail-app-password" -w
# 應輸出你的 App Password
```

### 步驟 4：設定 himalaya Gmail 設定

建立設定檔 `~/.config/himalaya/config.toml`：

```bash
mkdir -p ~/.config/himalaya
```

```toml
# ~/.config/himalaya/config.toml

[accounts.gmail]
email = "你的Gmail@gmail.com"
display-name = "你的名稱"
default = true

# IMAP 讀取郵件
backend.type = "imap"
backend.host = "imap.gmail.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "你的Gmail@gmail.com"
backend.auth.type = "password"
backend.auth.cmd = "security find-generic-password -a '你的Gmail@gmail.com' -s 'gmail-app-password' -w"

# SMTP 發送郵件
message.send.backend.type = "smtp"
message.send.backend.host = "smtp.gmail.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "你的Gmail@gmail.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "security find-generic-password -a '你的Gmail@gmail.com' -s 'gmail-app-password' -w"

# 簽名檔
signature = "Best regards"
signature-delim = "-- \n"

# 附件下載目錄
downloads-dir = "~/Downloads/himalaya"
```

### 步驟 5：驗證 himalaya Gmail 連線

```bash
# 測試列出收件箱
himalaya envelope list

# 測試讀取最新一封郵件
himalaya message read 1

# 測試發送郵件（確認 SMTP 正常）
himalaya message write <<'EOF'
From: 你的Gmail@gmail.com
To: 你自己的Gmail@gmail.com
Subject: CoPaw himalaya 測試

這是一封來自 CoPaw himalaya 的測試郵件。
EOF
```

### 步驟 6：建立支援發送的自訂 himalaya Skill

> **重要**：CoPaw 內建的 himalaya Skill 出於安全考量**預設停用郵件發送功能**。為了實現自動發送 Gmail，我們需要建立一個自訂版本的 Skill 來覆蓋內建版本。

```bash
mkdir -p ~/.copaw/customized_skills/himalaya
```

建立 `~/.copaw/customized_skills/himalaya/SKILL.md`：

```markdown
---
name: himalaya
description: "管理 Gmail 郵件 - 列出、讀取、搜尋、撰寫、回覆、轉寄郵件，支援附件處理"
---

# Himalaya Gmail 郵件管理

使用 himalaya CLI 管理 Gmail 郵件。

## 可用操作

### 列出郵件

```bash
himalaya envelope list
himalaya envelope list --folder "Sent"
himalaya envelope list --page 1 --page-size 20
```

### 搜尋郵件

```bash
himalaya envelope list from sender@example.com subject 關鍵字
```

### 讀取郵件

```bash
himalaya message read <郵件ID>
```

### 撰寫並發送郵件

使用 MML（MIME Meta Language）格式撰寫郵件：

```bash
himalaya message write <<'ENDMAIL'
From: 你的名稱 <你的Gmail@gmail.com>
To: 收件人 <recipient@example.com>
Subject: 郵件主題

郵件內文。
ENDMAIL
```

### 回覆郵件

```bash
himalaya message reply <郵件ID> <<'ENDMAIL'
回覆內容
ENDMAIL
```

### 轉寄郵件

```bash
himalaya message forward <郵件ID> <<'ENDMAIL'
To: forwarded@example.com

轉寄附加說明
ENDMAIL
```

### 管理附件

```bash
# 下載附件
himalaya attachment download <郵件ID>
himalaya attachment download <郵件ID> --dir ~/Downloads
```

### 移動/複製/刪除

```bash
himalaya message move <郵件ID> "Archive"
himalaya message copy <郵件ID> "Important"
himalaya message delete <郵件ID>
```

### 管理標記

```bash
himalaya flag add <郵件ID> --flag seen
himalaya flag remove <郵件ID> --flag seen
```

## 注意事項

- 發送郵件前務必確認收件人地址正確
- 附件使用 MML 語法附加
- 用 `--output json` 取得結構化輸出
- 若指令出錯，使用 `RUST_LOG=debug` 除錯
```

啟用自訂 Skill（覆蓋內建版本）：

```bash
copaw skills enable himalaya
```

驗證啟用狀態：

```bash
copaw skills list
# himalaya 應顯示為 "enabled"（來源為 customized）
```

---

## 第五部分：設定 GitHub 專案自動上傳

CoPaw 的 Agent 可以透過內建的 `execute_shell_command` 工具執行 Git 和 GitHub CLI 命令。不需要額外安裝任何 Skill。

### 步驟 1：設定 Git 全域設定

```bash
git config --global user.name "你的名稱"
git config --global user.email "你的Gmail@gmail.com"
```

### 步驟 2：認證 GitHub CLI

```bash
gh auth login
# 選擇 GitHub.com
# 選擇 HTTPS
# 選擇 Login with a web browser
# 按照指示完成認證
```

確認認證成功：

```bash
gh auth status
```

### 步驟 3：設定 SSH Key（可選，推薦）

```bash
# 產生 SSH Key
ssh-keygen -t ed25519 -C "你的Gmail@gmail.com"

# 新增到 SSH agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# 新增到 GitHub
gh ssh-key add ~/.ssh/id_ed25519.pub --title "Mac Mini M4"

# 測試連線
ssh -T git@github.com
```

### 步驟 4：告訴 Agent 你的 GitHub 資訊

編輯 `~/.copaw/PROFILE.md`，加入 GitHub 相關資訊：

```markdown
## 開發環境

- GitHub 帳號：你的GitHub用戶名
- 常用程式語言：Python, TypeScript
- 專案目錄：~/projects/
- Git 設定：已設定 SSH key 和 GitHub CLI

## 專案規範

- Commit message 使用 conventional commits（feat:、fix:、docs: 等）
- 專案應包含 README.md、.gitignore
- Python 專案使用 pyproject.toml
```

---

## 第六部分：設定自動化工作流程

### 工作流程 1：定時檢查 Gmail 並透過 Discord 通知

建立 Cron 任務，每 30 分鐘檢查 Gmail 新郵件並透過 Discord 通知：

```bash
copaw cron create \
  --type agent \
  --name "Gmail 新郵件檢查" \
  --cron "*/30 * * * *" \
  --channel discord \
  --target-user "你的Discord用戶ID" \
  --target-session "gmail_check" \
  --text "請檢查 Gmail 收件箱是否有新的未讀郵件。使用 himalaya envelope list 列出最近的郵件，如果有未讀郵件，請摘要每封郵件的寄件人、主題和關鍵內容，以清楚的格式回報給我。如果沒有新郵件，簡短回報即可。"
```

> **取得 Discord 用戶 ID**：在 Discord 中開啟「開發者模式」（使用者設定 → 進階 → 開發者模式），對自己的頭像按右鍵 → 「複製使用者 ID」。

### 工作流程 2：透過 Discord 指令建立專案並上傳 GitHub

這不需要額外設定 Cron，直接在 Discord 與 Bot 對話即可。以下是範例對話流程：

```
你：幫我建立一個 Python CLI 專案叫做 "hello-world"，
    功能是一個簡單的待辦事項管理工具，
    完成後上傳到我的 GitHub。

Agent 會自動：
1. mkdir -p ~/projects/hello-world
2. 建立 pyproject.toml、README.md、.gitignore
3. 撰寫程式碼
4. git init && git add . && git commit
5. gh repo create hello-world --public --source=. --push
6. 回報 GitHub 倉庫 URL
```

### 工作流程 3：完成後自動發送 Gmail 通知

你可以在 Discord 中直接告訴 Agent：

```
你：專案完成後，請發一封郵件到 teammate@example.com，
    告知專案已上傳至 GitHub，附上倉庫連結。
```

或者建立一個 Cron 定時任務，讓 Agent 每天下午 6 點自動彙報當天工作：

```bash
copaw cron create \
  --type agent \
  --name "每日工作彙報" \
  --cron "0 18 * * 1-5" \
  --channel discord \
  --target-user "你的Discord用戶ID" \
  --target-session "daily_report" \
  --text "請彙報今天的工作進度。檢查 ~/projects/ 目錄下今天有 git commit 的專案，整理成摘要。然後：1) 在 Discord 上告知我今天的工作摘要；2) 發送一封郵件到 你的Gmail@gmail.com，主題為「每日工作彙報 - [今天日期]」，內容包含今天的工作摘要和各專案的 GitHub 連結。"
```

### 工作流程 4：設定 Heartbeat 自動背景任務

Heartbeat 讓 Agent 定期自動執行背景任務。編輯 `~/.copaw/HEARTBEAT.md`：

```markdown
# 心跳任務

每次心跳觸發時，請依序執行以下任務：

1. **檢查 Gmail**：使用 `himalaya envelope list` 查看是否有新的未讀郵件
   - 如果有重要郵件（如 GitHub notification、客戶郵件），立即透過 Discord 通知
   - 將郵件摘要寫入今日的 memory 日記

2. **檢查專案狀態**：檢查 ~/projects/ 下各專案的 git status
   - 如果有未 commit 的變更，提醒我

3. **更新日記**：將今天處理的任務寫入 memory/YYYY-MM-DD.md
```

啟用 Heartbeat（每 2 小時一次，工作時段 08:00-22:00）：

**透過 API：**

```bash
curl -X PUT http://127.0.0.1:8088/api/config/heartbeat \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": true,
    "every": "2h",
    "target": "discord",
    "activeHours": {
      "start": "08:00",
      "end": "22:00"
    }
  }'
```

**或直接編輯 `~/.copaw/config.json`：**

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "enabled": true,
        "every": "2h",
        "target": "discord",
        "activeHours": {
          "start": "08:00",
          "end": "22:00"
        }
      }
    }
  }
}
```

---

## 第七部分：啟動與驗證

### 啟動 CoPaw 服務

```bash
# 前景啟動（開發/除錯用）
copaw app

# 背景啟動
nohup copaw app > ~/copaw.log 2>&1 &

# 檢查運行狀態
curl http://127.0.0.1:8088/api/version
```

### 完整驗證清單

啟動後，依序驗證各功能：

#### ✅ 驗證 1：Web Console

在瀏覽器開啟 `http://127.0.0.1:8088`，應看到 CoPaw 的聊天介面。

#### ✅ 驗證 2：AI 引擎

在 Console 中輸入「你好，請自我介紹」，Agent 應正常回覆。

#### ✅ 驗證 3：Discord Bot

在 Discord 中向 Bot 發送私訊「你好」，Bot 應回覆。

#### ✅ 驗證 4：Gmail 讀取

在 Discord 中告訴 Bot：

```
請檢查我的 Gmail 收件箱，列出最近 5 封郵件的摘要
```

#### ✅ 驗證 5：Gmail 發送

在 Discord 中告訴 Bot：

```
請發一封測試郵件到 你自己的Gmail@gmail.com，主題是「CoPaw 測試」，內容是「這是一封來自 CoPaw 的自動測試郵件。」
```

#### ✅ 驗證 6：GitHub 操作

在 Discord 中告訴 Bot：

```
請在 ~/projects/test-copaw 建立一個簡單的 Python Hello World 專案，然後上傳到我的 GitHub
```

#### ✅ 驗證 7：Cron 定時任務

```bash
# 確認 Cron 任務已建立
copaw cron list

# 手動執行一次測試
copaw cron run <job_id>
```

---

## 第八部分：進階設定與維運

### 使用 Supervisor 保持 CoPaw 長期運行

```bash
brew install supervisor
```

建立設定 `/usr/local/etc/supervisor.d/copaw.ini`：

```ini
[program:copaw]
command=/Users/你的使用者名稱/copaw-env/bin/copaw app
directory=/Users/你的使用者名稱
autostart=true
autorestart=true
stderr_logfile=/Users/你的使用者名稱/copaw-error.log
stdout_logfile=/Users/你的使用者名稱/copaw.log
environment=HOME="/Users/你的使用者名稱",PATH="/usr/local/bin:/usr/bin:/bin:/Users/你的使用者名稱/copaw-env/bin"
```

```bash
# 啟動 Supervisor
brew services start supervisor

# 管理 CoPaw 進程
supervisorctl start copaw
supervisorctl status copaw
supervisorctl restart copaw
```

### 使用 launchd 開機自動啟動（macOS 原生方式）

建立 `~/Library/LaunchAgents/com.copaw.agent.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.copaw.agent</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/你的使用者名稱/copaw-env/bin/copaw</string>
        <string>app</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>WorkingDirectory</key>
    <string>/Users/你的使用者名稱</string>
    <key>StandardOutPath</key>
    <string>/Users/你的使用者名稱/copaw.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/你的使用者名稱/copaw-error.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin:/Users/你的使用者名稱/copaw-env/bin</string>
    </dict>
</dict>
</plist>
```

```bash
# 載入並啟動
launchctl load ~/Library/LaunchAgents/com.copaw.agent.plist

# 停止
launchctl unload ~/Library/LaunchAgents/com.copaw.agent.plist
```

### 設定長期記憶（Embedding）

若需要 Agent 具備長期記憶搜尋能力，需設定 Embedding API：

```bash
# 使用 DashScope Embedding（阿里雲免費額度）
copaw env set EMBEDDING_API_KEY "sk-你的DashScope-Key"
copaw env set EMBEDDING_MODEL_NAME "text-embedding-v4"
copaw env set EMBEDDING_DIMENSIONS "1024"

# 啟用記憶管理
copaw env set ENABLE_MEMORY_MANAGER "true"

# macOS 使用 chroma 後端（預設）
copaw env set MEMORY_STORE_BACKEND "auto"
```

### 備份與還原

```bash
# 備份整個工作目錄
curl -o copaw-backup.zip http://127.0.0.1:8088/api/workspace/download

# 還原
curl -X POST http://127.0.0.1:8088/api/workspace/upload \
  -F "file=@copaw-backup.zip"
```

---

## 常見問題排除

### Q: Ollama 模型回應太慢

**原因**：16B 或更大模型超出 M4 16GB 的舒適範圍。

**解法**：
```bash
# 使用更小的模型
ollama pull qwen3:4b

# 或使用量化更高的版本
ollama pull qwen3:8b-q4_0

# 或改用遠端 Ollama（見方案 C），讓 24GB 機器跑大模型
```

### Q: 遠端 Ollama 連線失敗

**檢查清單**：

1. **24GB 機器 Ollama 是否運行**：在 24GB 機器上執行 `curl http://localhost:11434/api/tags`
2. **OLLAMA_HOST 是否已設定**：24GB 機器需設定 `OLLAMA_HOST=0.0.0.0:11434` 並重啟服務
3. **網路連通性**：在 16GB 機器上執行 `curl http://24GB機器IP:11434/api/tags`
4. **防火牆**：24GB 機器需放行 11434 埠
5. **base_url 格式**：需包含 `/v1`，例如 `http://192.168.1.100:11434/v1`
6. **IP 是否變動**：若使用 DHCP，建議在路由器設定 DHCP 保留

**除錯**：
```bash
# 在 16GB 機器上測試
curl -v http://192.168.1.100:11434/api/tags

# 若跨網段，可先用 SSH 隧道測試
ssh -L 11434:localhost:11434 user@24gb-ip -N &
curl http://localhost:11434/api/tags
```

### Q: Discord Bot 無回應

**檢查清單**：
1. Bot Token 是否正確
2. Bot 是否已加入伺服器
3. Message Content Intent 是否已開啟
4. CoPaw 服務是否正在運行：`curl http://127.0.0.1:8088/api/version`
5. 查看日誌：`tail -f ~/copaw.log`

### Q: himalaya 無法連接 Gmail

**檢查清單**：
1. 確認 App Password 正確：`security find-generic-password -a "your@gmail.com" -s "gmail-app-password" -w`
2. 確認兩步驗證已啟用
3. 手動測試：`himalaya envelope list`
4. 開啟 debug：`RUST_LOG=debug himalaya envelope list`

### Q: GitHub push 失敗

**檢查清單**：
1. `gh auth status` 是否已認證
2. `ssh -T git@github.com` 是否成功
3. 確認 Agent 工作目錄有權限：`ls -la ~/projects/`

### Q: Claude API 連線失敗

**可能原因**：CoPaw 使用 OpenAI 相容介面呼叫 Anthropic API，部分版本可能不完全相容。

**替代方案**：使用 LiteLLM 作為 OpenAI 相容 Proxy：

```bash
pip install litellm
litellm --model claude-sonnet-4-20250514 --port 4000 --api_key sk-ant-你的Key &
```

然後在 CoPaw 中建立自訂 Provider，`base_url` 指向 `http://localhost:4000/v1`。

### Q: Cron 任務沒有觸發

**檢查**：
```bash
# 列出任務
copaw cron list

# 檢查任務狀態
copaw cron state <job_id>

# 手動執行一次
copaw cron run <job_id>
```

確認 CoPaw 服務在 Cron 排定時間持續運行中。

---

## 完整設定檔參考

### `~/.copaw/config.json` 完整範例

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "bot_token": "你的Discord Bot Token",
      "http_proxy": "",
      "http_proxy_auth": "",
      "bot_prefix": ""
    },
    "console": {
      "enabled": true
    },
    "telegram": { "enabled": false, "bot_token": "" },
    "dingtalk": { "enabled": false, "client_id": "", "client_secret": "" },
    "feishu": { "enabled": false, "app_id": "", "app_secret": "" },
    "qq": { "enabled": false, "app_id": "", "client_secret": "" },
    "imessage": { "enabled": false }
  },
  "mcp": {
    "clients": {}
  },
  "agents": {
    "language": "zh",
    "running": {
      "max_iters": 50,
      "max_input_length": 131072
    },
    "defaults": {
      "heartbeat": {
        "enabled": true,
        "every": "2h",
        "target": "discord",
        "activeHours": {
          "start": "08:00",
          "end": "22:00"
        }
      }
    }
  },
  "show_tool_details": true
}
```

### `~/.copaw/providers.json` 完整範例（雙引擎）

```json
{
  "providers": {
    "ollama": {
      "base_url": "http://localhost:11434/v1",
      "api_key": "",
      "extra_models": []
    }
  },
  "custom_providers": {
    "anthropic": {
      "id": "anthropic",
      "name": "Anthropic (Claude)",
      "default_base_url": "https://api.anthropic.com/v1",
      "api_key_prefix": "sk-ant-",
      "models": [
        { "id": "claude-sonnet-4-20250514", "name": "Claude Sonnet 4" },
        { "id": "claude-opus-4-20250514", "name": "Claude Opus 4" },
        { "id": "claude-3-5-haiku-20241022", "name": "Claude 3.5 Haiku" }
      ],
      "base_url": "https://api.anthropic.com/v1",
      "api_key": "sk-ant-你的API Key",
      "chat_model": "OpenAIChatModel"
    }
  },
  "active_llm": {
    "provider_id": "anthropic",
    "model": "claude-sonnet-4-20250514"
  }
}
```

### `~/.copaw/providers.json` 完整範例（三引擎：本地 + 遠端 Ollama + Claude）

適用於 16GB 本機 + 24GB 遠端 Ollama 的混合架構：

```json
{
  "providers": {
    "ollama": {
      "base_url": "http://localhost:11434/v1",
      "api_key": "",
      "extra_models": []
    }
  },
  "custom_providers": {
    "ollama-remote": {
      "id": "ollama-remote",
      "name": "Ollama 遠端 (24GB)",
      "default_base_url": "http://192.168.1.100:11434/v1",
      "api_key_prefix": "",
      "models": [
        { "id": "qwen3:14b", "name": "Qwen3 14B" },
        { "id": "qwen2.5-coder:14b", "name": "Qwen 2.5 Coder 14B" },
        { "id": "qwen3:32b", "name": "Qwen3 32B" }
      ],
      "base_url": "http://192.168.1.100:11434/v1",
      "api_key": "",
      "chat_model": "OpenAIChatModel"
    },
    "anthropic": {
      "id": "anthropic",
      "name": "Anthropic (Claude)",
      "default_base_url": "https://api.anthropic.com/v1",
      "api_key_prefix": "sk-ant-",
      "models": [
        { "id": "claude-sonnet-4-20250514", "name": "Claude Sonnet 4" },
        { "id": "claude-opus-4-20250514", "name": "Claude Opus 4" },
        { "id": "claude-3-5-haiku-20241022", "name": "Claude 3.5 Haiku" }
      ],
      "base_url": "https://api.anthropic.com/v1",
      "api_key": "sk-ant-你的API Key",
      "chat_model": "OpenAIChatModel"
    }
  },
  "active_llm": {
    "provider_id": "ollama-remote",
    "model": "qwen3:14b"
  }
}
```

> **注意**：將 `192.168.1.100` 替換為你的 24GB 機器實際 IP。`ollama-remote.models` 需與遠端 `ollama list` 的模型一致。

### `~/.config/himalaya/config.toml` 完整範例

```toml
[accounts.gmail]
email = "你的Gmail@gmail.com"
display-name = "你的名稱"
default = true

backend.type = "imap"
backend.host = "imap.gmail.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "你的Gmail@gmail.com"
backend.auth.type = "password"
backend.auth.cmd = "security find-generic-password -a '你的Gmail@gmail.com' -s 'gmail-app-password' -w"

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.gmail.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "你的Gmail@gmail.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "security find-generic-password -a '你的Gmail@gmail.com' -s 'gmail-app-password' -w"

signature = "Best regards"
signature-delim = "-- \n"
downloads-dir = "~/Downloads/himalaya"
```

### `~/.copaw/HEARTBEAT.md` 完整範例

```markdown
# 心跳任務

每次心跳觸發時，請依序執行以下任務：

1. **檢查 Gmail 新郵件**
   - 使用 `himalaya envelope list` 列出最新郵件
   - 若有未讀重要郵件，透過 Discord 通知我摘要

2. **檢查 GitHub 專案狀態**
   - 檢查 ~/projects/ 下各專案的 `git status`
   - 若有未提交的變更，提醒我

3. **更新日記**
   - 將本次檢查結果寫入 memory/YYYY-MM-DD.md
```

### `~/.copaw/PROFILE.md` 完整範例

```markdown
# Agent 身份

- 名稱：Friday
- 語言：繁體中文

# 用戶資料

- 名稱：（你的名稱）
- 主要語言：繁體中文
- 工作時間：週一至週五 09:00-18:00
- 時區：Asia/Taipei

## 開發環境

- 設備：Mac Mini M4 16GB
- GitHub 帳號：（你的GitHub用戶名）
- 常用程式語言：Python, TypeScript
- 專案目錄：~/projects/
- Git 設定：已設定 SSH key 和 GitHub CLI

## 聯絡方式

- Email：你的Gmail@gmail.com
- Discord：（你的Discord用戶名）

## 偏好

- 回覆風格：簡潔直接
- 程式碼風格：conventional commits、有 README
- 通知偏好：重要事項透過 Discord DM 通知
```
