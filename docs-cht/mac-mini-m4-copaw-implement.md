# Mac Mini M4 (16GB) CoPaw 完整實作指南

> 本指南詳細說明如何在 **Mac Mini M4 (16GB)** 上從無到有部署 CoPaw，整合 Ollama 本地模型、Claude API、Discord 機器人、Gmail 自動收發、GitHub 專案上傳與通知。16GB 記憶體可舒適運行 7B ~ 8B 本地模型，並可透過遠端 Ollama（連接 24GB 機器）使用更大的 14B ~ 32B 模型。
>
> 若你擁有 Mac Mini M4 **24GB**，請參考 [24GB 版指南](./mac-mini-m4-24g-copaw-implement.md)，可運行更大模型並支援本地 Embedding。

---

## 目錄

- [前置環境準備](#前置環境準備)
- [第一部分：安裝 CoPaw](#第一部分安裝-copaw)
- [第二部分：設定 AI 引擎](#第二部分設定-ai-引擎)
  - [方案 A：Ollama 本地模型（16GB 攻略）](#方案-aollama-本地模型16gb-攻略)
  - [方案 B：Claude API（Anthropic）](#方案-bclaude-apianthropic)
  - [方案 C：遠端 Ollama（連接 24GB 機器）](#方案-c遠端-ollama連接-24gb-機器)
  - [方案 D：混合架構（推薦）](#方案-d混合架構推薦)
- [第三部分：16GB 模型選擇指南](#第三部分16gb-模型選擇指南)
- [第四部分：記憶體效能優化](#第四部分記憶體效能優化)
- [第五部分：設定 Discord 機器人](#第五部分設定-discord-機器人)
- [第六部分：設定 Gmail 自動收發](#第六部分設定-gmail-自動收發)
- [第七部分：設定 GitHub 專案自動上傳](#第七部分設定-github-專案自動上傳)
- [第八部分：設定自動化工作流程](#第八部分設定自動化工作流程)
- [第九部分：長期記憶設定（雲端 Embedding）](#第九部分長期記憶設定雲端-embedding)
- [第十部分：啟動與驗證](#第十部分啟動與驗證)
- [第十一部分：進階設定與維運](#第十一部分進階設定與維運)
- [常見問題排除](#常見問題排除)
- [完整設定檔參考](#完整設定檔參考)

---

## 前置環境準備

### 硬體確認

Mac Mini M4 16GB 採用統一記憶體架構（Unified Memory），CPU 與 GPU 共享同一塊高頻寬記憶體，搭配 Metal GPU 加速，可運行 7B ~ 8B 本地 LLM。

### 記憶體規劃

```
16GB 總記憶體
├── macOS 系統          ~3-4 GB
├── CoPaw 服務          ~0.5 GB
├── 瀏覽器/其他應用      ~2-3 GB
├── 主力 LLM 模型       ~4-5 GB（7B ~ 8B 模型）
└── 緩衝空間            ~3-5 GB
```

> **注意**：16GB 記憶體較為緊張，**不建議**同時載入多個模型或運行本地 Embedding。建議使用雲端 Embedding API，或透過遠端 Ollama 連接 24GB 機器使用更大模型。

### 與 24GB 的差異

| 項目 | 16GB | 24GB |
|------|------|------|
| **可用模型範圍** | 7B ~ 8B（14B 勉強） | 7B ~ 32B（Q4 量化） |
| **推薦主力模型** | Qwen3 8B | Qwen3 14B / 32B（Q4） |
| **並行推理** | 不建議 | 可同時載入 2 個模型 |
| **本地 Embedding** | 不建議（用雲端 API） | 可運行本地 Embedding 模型 |
| **MLX 後端** | 受限（僅小模型） | 完整支援，效能更優 |
| **Ollama 並行數** | 1 | 2 |
| **遠端 Ollama** | 推薦（使用 24GB 機器） | 可作為伺服器 |

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

> **16GB 注意**：不建議安裝 `mlx` extra，MLX 在 16GB 下效益有限，使用 Ollama 即可。

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

16GB 記憶體下有四種方案可以組合使用。推薦方案 D（混合架構）。

### 方案 A：Ollama 本地模型（16GB 攻略）

Ollama 可讓你在 Mac Mini M4 上離線運行開源 LLM，資料不外傳。16GB 可舒適運行 7B ~ 8B 模型。

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

#### 步驟 A3：下載 16GB 適用模型

16GB 建議使用 7B ~ 8B 模型，14B 可勉強運行但記憶體較為緊張。

```bash
# ═══════════════════════════════════════════
# 🏆 推薦首選
# ═══════════════════════════════════════════

# Qwen3 8B — 中英文兼優、推理能力強（~5GB 記憶體）
ollama pull qwen3:8b

# Qwen3 4B — 極速、適合簡單任務（~2.5GB）
ollama pull qwen3:4b


# ═══════════════════════════════════════════
# 💻 程式開發專用
# ═══════════════════════════════════════════

# Qwen 2.5 Coder 7B — 程式撰寫能力優秀（~4.5GB）
ollama pull qwen2.5-coder:7b


# ═══════════════════════════════════════════
# ⚡ 其他推薦
# ═══════════════════════════════════════════

# Llama 3.1 8B — 英文能力優秀（~5GB）
ollama pull llama3.1:8b

# Gemma 3 4B — Google 多模態輕量模型（~3GB）
ollama pull gemma3:4b

# Mistral 7B — 輕量快速（~4.5GB）
ollama pull mistral:7b

# Qwen3 30B-A3B（MoE）— 30B 參數但僅啟用 3B（~2GB，速度極快）
ollama pull qwen3:30b-a3b


# ═══════════════════════════════════════════
# 🔥 挑戰 14B（記憶體緊張，謹慎使用）
# ═══════════════════════════════════════════

# Qwen3 14B — 需 ~8.5GB，加上系統後接近極限
ollama pull qwen3:14b
# ⚠ 載入 14B 後記憶體剩餘很少
# 建議僅在不運行瀏覽器等大型應用時使用
# 更好的方式是使用遠端 Ollama（方案 C）
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
# 選擇 model: qwen3:8b（推薦）
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

#### 步驟 A5：Ollama 效能調校（16GB 專屬）

16GB 記憶體較為緊張，需要保守設定：

```bash
# ═══════════════════════════════════════════
# 16GB 專屬 Ollama 環境變數
# ═══════════════════════════════════════════

# 限制 1 個並行請求（節省記憶體）
export OLLAMA_NUM_PARALLEL=1

# 模型用完後 10 分鐘自動卸載（釋放記憶體）
export OLLAMA_KEEP_ALIVE=10m

# 最多同時載入 1 個模型
export OLLAMA_MAX_LOADED_MODELS=1

# GPU 層數最大化（M4 Metal 加速）
export OLLAMA_GPU_LAYERS=999

# 加入 ~/.zshrc 使其永久生效
cat >> ~/.zshrc << 'ZSHRC'
export OLLAMA_NUM_PARALLEL=1
export OLLAMA_KEEP_ALIVE=10m
export OLLAMA_MAX_LOADED_MODELS=1
export OLLAMA_GPU_LAYERS=999
ZSHRC

source ~/.zshrc
```

**參數說明：**

| 環境變數 | 16GB 建議值 | 說明 |
|----------|------------|------|
| `OLLAMA_NUM_PARALLEL` | `1` | 只允許 1 個並行請求（節省記憶體） |
| `OLLAMA_KEEP_ALIVE` | `10m` | 模型閒置 10 分鐘後自動卸載 |
| `OLLAMA_MAX_LOADED_MODELS` | `1` | 最多 1 個模型（避免記憶體溢出） |
| `OLLAMA_GPU_LAYERS` | `999` | 所有層都用 GPU（M4 Metal 加速） |

---

### 方案 B：Claude API（Anthropic）

CoPaw **沒有內建 Anthropic Provider**，需要建立自訂 Provider。Claude 的優勢在於推理能力強、中文品質高、**不佔本地記憶體**，非常適合 16GB 機器。

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

> **注意**：CoPaw 使用 OpenAI 相容介面（`OpenAIChatModel`）呼叫 Claude API。若遇到問題，可嘗試使用第三方 OpenAI 相容 proxy（如 LiteLLM）。

#### 步驟 B3：驗證 Claude 連線

```bash
# 測試 Provider 連線（需先啟動 CoPaw）
copaw app &
curl -X POST http://127.0.0.1:8088/api/models/anthropic/test
```

---

### 方案 C：遠端 Ollama（連接 24GB 機器）

> **16GB 用戶的最佳擴展方案**：若你同時擁有 Mac Mini M4 24GB，可以讓 16GB 的 CoPaw 透過網路呼叫 24GB 機器上的 Ollama，使用 14B ~ 32B 等更大模型，完全不佔用本機記憶體。

#### 架構說明

```
┌─────────────────────────────────┐    網路 (LAN)    ┌─────────────────────────────────┐
│  Mac Mini M4 16GB（本機 CoPaw）  │ ◄─────────────► │  Mac Mini M4 24GB（遠端 Ollama）  │
│                                 │   HTTP :11434   │                                 │
│  • CoPaw 服務                    │                 │  • Ollama 服務                    │
│  • Discord / Gmail / GitHub      │                 │  • Qwen3 14B / 32B 等大模型       │
│  • 本地 Ollama（可選，7B~8B）    │                 │  • 本地 Embedding（可選）         │
└─────────────────────────────────┘                 └─────────────────────────────────┘
```

#### 前提條件

24GB 機器需要：
1. 已安裝 Ollama 並下載 14B+ 模型
2. 設定 `OLLAMA_HOST=0.0.0.0:11434` 以接受遠端連線
3. 防火牆放行 11434 埠

> **24GB 機器的完整遠端伺服器設定**請參考 [24GB 版指南 — 附錄 A：作為遠端 Ollama 伺服器](./mac-mini-m4-24g-copaw-implement.md#附錄-a作為遠端-ollama-伺服器)。

#### 步驟 C1：確認 24GB 機器可連線

```bash
# 取得 24GB 機器的區網 IP（在 24GB 機器上執行）
ifconfig | grep "inet " | grep -v 127.0.0.1
# 例如：192.168.1.100

# 在 16GB 本機上測試連線
curl http://192.168.1.100:11434/api/tags
# 應回傳 JSON，列出遠端已下載的模型
```

#### 步驟 C2：在 CoPaw 建立遠端 Ollama Provider

**方式一：CLI 互動設定**

```bash
copaw models add-provider ollama-remote \
  -n "Ollama 遠端 (24GB)" \
  -u "http://192.168.1.100:11434/v1" \
  --api-key-prefix ""

# 新增遠端模型（需與 24GB 機器上已下載的模型一致）
copaw models add-model ollama-remote -m "qwen3:14b" -n "Qwen3 14B"
copaw models add-model ollama-remote -m "qwen2.5-coder:14b" -n "Qwen 2.5 Coder 14B"
copaw models add-model ollama-remote -m "qwen3:32b" -n "Qwen3 32B"
copaw models add-model ollama-remote -m "qwen3:30b-a3b" -n "Qwen3 30B-A3B MoE"

# 設為啟用模型
copaw models set-llm
# 選擇 ollama-remote → qwen3:14b
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
  "custom_providers": {
    "ollama-remote": {
      "id": "ollama-remote",
      "name": "Ollama 遠端 (24GB)",
      "default_base_url": "http://192.168.1.100:11434/v1",
      "api_key_prefix": "",
      "models": [
        { "id": "qwen3:14b", "name": "Qwen3 14B" },
        { "id": "qwen2.5-coder:14b", "name": "Qwen 2.5 Coder 14B" },
        { "id": "qwen3:32b", "name": "Qwen3 32B" },
        { "id": "qwen3:30b-a3b", "name": "Qwen3 30B-A3B MoE" }
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

> **重要**：`models` 陣列需與 24GB 機器上實際已下載的模型一致。在 24GB 機器上執行 `ollama list` 確認。

#### 步驟 C3：使用遠端 Embedding（可選）

若 24GB 機器有運行 `nomic-embed-text`，可以使用遠端 Embedding 啟用長期記憶：

```bash
copaw env set EMBEDDING_BASE_URL "http://192.168.1.100:11434/v1"
copaw env set EMBEDDING_API_KEY "ollama"
copaw env set EMBEDDING_MODEL_NAME "nomic-embed-text"
copaw env set EMBEDDING_DIMENSIONS "768"
copaw env set ENABLE_MEMORY_MANAGER "true"
```

#### 網路與安全考量

| 情境 | 建議 |
|------|------|
| **同一區域網路（LAN）** | 使用區網 IP（如 `192.168.1.100`），延遲低、設定簡單 |
| **不同網段 / VPN** | 確保 11434 埠可達，或透過 SSH 隧道 |
| **暴露於網際網路** | ⚠ 不建議。Ollama 無內建認證 |
| **固定 IP** | 建議在路由器為 24GB 機器設定 DHCP 保留 |

**SSH 隧道（跨網段 / 安全連線）**：

```bash
# 在本機（16GB）上執行，建立隧道
ssh -L 11434:localhost:11434 user@24gb-machine-ip -N &

# 然後 CoPaw 的 base_url 設為 http://localhost:11434/v1
# 流量會經由 SSH 加密轉發到 24GB 機器
```

---

### 方案 D：混合架構（推薦）

16GB 的最佳實踐是**混合使用本地小模型、遠端大模型與雲端 API**：

```
日常對話 / 簡單任務  →  本地 Ollama Qwen3 8B（免費，離線可用）
複雜對話 / 深度推理  →  遠端 Ollama Qwen3 14B（免費，需 24GB 機器在線）
程式開發 / 程式碼    →  遠端 Ollama Qwen2.5-Coder 14B（免費）
極複雜推理 / 長文    →  Claude Sonnet 4（雲端，按量付費）
Embedding / 記憶    →  DashScope API（雲端）或遠端 Ollama
```

**切換引擎：**

```bash
copaw models set-llm
# 選 ollama → qwen3:8b                           # 本地簡單任務
# 選 ollama-remote → qwen3:14b                   # 遠端大模型
# 選 ollama-remote → qwen2.5-coder:14b           # 遠端程式開發
# 選 anthropic → claude-sonnet-4-20250514         # 雲端複雜任務
```

也可以透過 Discord 告訴 Agent：

```
你：接下來的任務比較複雜，請幫我切換到 Claude
```

---

## 第三部分：16GB 模型選擇指南

### Ollama 本地模型推薦

#### 綜合對話模型

| 模型 | 參數量 | 記憶體用量 | 推理速度 | 中文能力 | 推薦度 |
|------|--------|-----------|---------|---------|--------|
| `qwen3:8b` | 8B | ~5 GB | ~45 t/s | ★★★★☆ | ⭐⭐⭐ 最推薦 |
| `qwen3:30b-a3b` | 30B MoE | ~2 GB | ~60 t/s | ★★★★☆ | ⭐⭐ 極速 |
| `qwen3:4b` | 4B | ~2.5 GB | ~65 t/s | ★★★☆☆ | ⭐⭐ 極輕量 |
| `llama3.1:8b` | 8B | ~5 GB | ~45 t/s | ★★☆☆☆ | 英文專用 |
| `gemma3:4b` | 4B | ~3 GB | ~55 t/s | ★★★☆☆ | 多模態輕量 |
| `mistral:7b` | 7B | ~4.5 GB | ~50 t/s | ★★☆☆☆ | 輕量 |
| `qwen3:14b` | 14B | ~8.5 GB | ~30 t/s | ★★★★★ | ⚠ 挑戰極限 |

#### 程式開發模型

| 模型 | 參數量 | 記憶體用量 | 推理速度 | 程式能力 | 推薦度 |
|------|--------|-----------|---------|---------|--------|
| `qwen2.5-coder:7b` | 7B | ~4.5 GB | ~50 t/s | ★★★☆☆ | ⭐⭐⭐ 推薦 |
| `qwen2.5-coder:3b` | 3B | ~2 GB | ~70 t/s | ★★☆☆☆ | 極速 |
| `qwen2.5-coder:14b` | 14B | ~8.5 GB | ~30 t/s | ★★★★★ | ⚠ 挑戰極限 |

### 16GB 模型組合建議

#### 組合 1：本地節約組合（推薦）

```bash
ollama pull qwen3:8b            # 主力對話 (~5GB)
ollama pull qwen2.5-coder:7b    # 程式開發 (~4.5GB)
# 不同時載入，按需切換
```

#### 組合 2：極速組合

```bash
ollama pull qwen3:30b-a3b       # MoE 極速對話 (~2GB)
ollama pull qwen3:4b            # 備用輕量 (~2.5GB)
ollama pull qwen2.5-coder:3b    # 快速程式 (~2GB)
# 記憶體非常充裕
```

#### 組合 3：本地 + 遠端（最佳體驗）

```bash
# 本地（16GB）
ollama pull qwen3:8b            # 離線備用 (~5GB)

# 遠端（24GB 機器）— 不佔本機記憶體
# qwen3:14b                    # 日常對話
# qwen2.5-coder:14b            # 程式開發
# Claude Sonnet 4              # 雲端複雜任務
```

---

## 第四部分：記憶體效能優化

> 16GB 記憶體較為緊張，以下優化策略對確保穩定運行至關重要。

### 4.1 macOS 記憶體壓力監控

```bash
# 即時監控記憶體（每秒更新）
vm_stat 1

# 檢查 Swap 使用量（理想值為 0）
sysctl vm.swapusage

# 檢查記憶體壓力等級
memory_pressure

# 使用 Activity Monitor（圖形化監控）
open -a "Activity Monitor"
```

### 4.2 Ollama 記憶體管理（16GB 保守策略）

16GB 下必須嚴格控制模型的記憶體使用：

```bash
# 模型用完後 5 分鐘自動卸載（節省記憶體）
export OLLAMA_KEEP_ALIVE=5m

# 絕對只載入 1 個模型
export OLLAMA_MAX_LOADED_MODELS=1

# 只允許 1 個並行請求
export OLLAMA_NUM_PARALLEL=1
```

#### 手動卸載模型（釋放記憶體）

```bash
# 強制卸載特定模型
curl http://localhost:11434/api/generate -d '{"model":"qwen3:8b","keep_alive":0}'

# 卸載所有模型
for model in $(ollama list | awk 'NR>1 {print $1}'); do
  curl -s http://localhost:11434/api/generate -d "{\"model\":\"$model\",\"keep_alive\":0}" > /dev/null
done
echo "所有模型已卸載"
```

### 4.3 macOS 系統層級優化

#### 關閉不必要的系統服務

```bash
# 減少 Spotlight 索引的記憶體佔用（16GB 建議關閉）
sudo mdutil -a -i off

# 需要時再開啟
# sudo mdutil -a -i on
```

#### 登入項目精簡

「系統設定 → 一般 → 登入項目與延伸功能」中，移除不必要的開機啟動項目。16GB 機器上這點尤為重要。

#### 減少瀏覽器記憶體佔用

使用 Ollama 時，建議：
- 關閉不需要的瀏覽器分頁
- 使用 Safari 取代 Chrome（記憶體效率更好）
- 或將瀏覽器完全關閉，讓更多記憶體給模型

### 4.4 CoPaw 層級優化

```bash
# 編輯 config.json 或透過 API
curl -X PUT http://127.0.0.1:8088/api/agent/running-config \
  -H "Content-Type: application/json" \
  -d '{
    "max_iters": 50,
    "max_input_length": 131072
  }'
```

| 設定 | 16GB 建議 | 說明 |
|------|----------|------|
| `max_iters` | 50 | 預設迭代次數 |
| `max_input_length` | 131072 | 128K tokens |
| `MEMORY_COMPACT_RATIO` | 0.7 | 較早觸發壓縮，節省記憶體 |
| `MEMORY_COMPACT_KEEP_RECENT` | 3 | 壓縮後保留 3 條最近對話 |

```bash
# 調整記憶壓縮參數（16GB 需要較早壓縮）
copaw env set COPAW_MEMORY_COMPACT_RATIO "0.7"
copaw env set COPAW_MEMORY_COMPACT_KEEP_RECENT "3"
```

### 4.5 Swap 監控

```bash
# 檢查 Swap
sysctl vm.swapusage

# 理想狀態：Swap 使用量為 0 或很小
# 如果 Swap 超過 1GB，表示記憶體不足，考慮：
# 1. 使用更小的模型（如 4B 而非 8B）
# 2. 關閉其他大型應用
# 3. 改用遠端 Ollama（方案 C）
```

### 4.6 記憶體監控腳本

```bash
cat > ~/check-memory.sh << 'SCRIPT'
#!/bin/bash
echo "=== Mac Mini M4 16GB 記憶體狀態 ==="
echo ""

# 系統記憶體壓力
echo "--- 記憶體壓力 ---"
memory_pressure
echo ""

# Swap 使用量
echo "--- Swap ---"
sysctl vm.swapusage
echo ""

# Ollama 佔用
echo "--- Ollama 進程 ---"
ps aux | grep -i ollama | grep -v grep | awk '{printf "PID: %s  RSS: %.0f MB  CMD: %s\n", $2, $6/1024, $11}'
echo ""

# CoPaw 佔用
echo "--- CoPaw 進程 ---"
ps aux | grep -i copaw | grep -v grep | awk '{printf "PID: %s  RSS: %.0f MB  CMD: %s\n", $2, $6/1024, $11}'
echo ""

# 已載入的 Ollama 模型
echo "--- Ollama 已載入模型 ---"
curl -s http://localhost:11434/api/ps 2>/dev/null | python3 -m json.tool 2>/dev/null || echo "Ollama 未運行"
SCRIPT

chmod +x ~/check-memory.sh
```

---

## 第五部分：設定 Discord 機器人

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

**方式二：CLI 互動設定**

```bash
copaw channels config
# 選擇 discord → 輸入 Bot Token
```

### 步驟 3：與 Bot 互動測試

1. 啟動 CoPaw：`copaw app`
2. 在 Discord 中向 Bot 發送 DM（私訊）或在頻道中 @Bot
3. Bot 應回覆，確認連線正常

---

## 第六部分：設定 Gmail 自動收發

CoPaw 使用 **himalaya CLI** 作為郵件客戶端。

### 步驟 1：安裝 himalaya CLI

```bash
brew install himalaya
himalaya --version
```

### 步驟 2：取得 Gmail App Password

1. 前往 [Google 帳戶安全性](https://myaccount.google.com/security)
2. 確認已啟用**兩步驗證**
3. 前往 [應用程式密碼](https://myaccount.google.com/apppasswords)
4. 選擇「郵件」與「Mac」，點擊**產生**
5. 複製產生的 16 位密碼（格式：`xxxx xxxx xxxx xxxx`）

### 步驟 3：安全儲存密碼

```bash
security add-generic-password \
  -a "你的Gmail@gmail.com" \
  -s "gmail-app-password" \
  -w "xxxx xxxx xxxx xxxx"
```

### 步驟 4：設定 himalaya Gmail 設定

```bash
mkdir -p ~/.config/himalaya
```

建立 `~/.config/himalaya/config.toml`：

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

### 步驟 5：驗證 himalaya Gmail 連線

```bash
himalaya envelope list
himalaya message read 1
```

### 步驟 6：建立支援發送的自訂 himalaya Skill

> **重要**：CoPaw 內建的 himalaya Skill 出於安全考量**預設停用郵件發送功能**。需要建立自訂版本。

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
himalaya attachment download <郵件ID>
himalaya attachment download <郵件ID> --dir ~/Downloads
```

## 注意事項

- 發送郵件前務必確認收件人地址正確
- 用 `--output json` 取得結構化輸出
```

啟用自訂 Skill：

```bash
copaw skills enable himalaya
```

---

## 第七部分：設定 GitHub 專案自動上傳

### 步驟 1：設定 Git 全域設定

```bash
git config --global user.name "你的名稱"
git config --global user.email "你的Gmail@gmail.com"
```

### 步驟 2：認證 GitHub CLI

```bash
gh auth login
```

### 步驟 3：設定 SSH Key（可選，推薦）

```bash
ssh-keygen -t ed25519 -C "你的Gmail@gmail.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
gh ssh-key add ~/.ssh/id_ed25519.pub --title "Mac Mini M4 16GB"
ssh -T git@github.com
```

### 步驟 4：告訴 Agent 你的 GitHub 資訊

編輯 `~/.copaw/PROFILE.md`，加入 GitHub 相關資訊。

---

## 第八部分：設定自動化工作流程

### 工作流程 1：定時檢查 Gmail 並透過 Discord 通知

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

### 工作流程 2：透過 Discord 建立專案並上傳 GitHub

直接在 Discord 與 Bot 對話：

```
你：幫我建立一個 Python CLI 專案叫做 "hello-world"，
    功能是一個簡單的待辦事項管理工具，
    完成後上傳到我的 GitHub。
```

### 工作流程 3：每日工作彙報

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

### 工作流程 4：Heartbeat 自動背景任務

編輯 `~/.copaw/HEARTBEAT.md` 並啟用（每 2 小時，08:00-22:00）：

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

---

## 第九部分：長期記憶設定（雲端 Embedding）

> **16GB 建議**：使用雲端 Embedding API，不佔用寶貴的本地記憶體。若有 24GB 遠端機器，也可使用遠端 Embedding。

### 方案 1：DashScope 雲端 Embedding（推薦）

```bash
copaw env set EMBEDDING_API_KEY "sk-你的DashScope-Key"
copaw env set EMBEDDING_MODEL_NAME "text-embedding-v4"
copaw env set EMBEDDING_DIMENSIONS "1024"
copaw env set ENABLE_MEMORY_MANAGER "true"
```

### 方案 2：遠端 Ollama Embedding（需 24GB 機器）

若 24GB 機器有運行 `nomic-embed-text`：

```bash
copaw env set EMBEDDING_BASE_URL "http://192.168.1.100:11434/v1"
copaw env set EMBEDDING_API_KEY "ollama"
copaw env set EMBEDDING_MODEL_NAME "nomic-embed-text"
copaw env set EMBEDDING_DIMENSIONS "768"
copaw env set ENABLE_MEMORY_MANAGER "true"
```

### 方案比較

| 指標 | DashScope 雲端 | 遠端 Ollama |
|------|---------------|------------|
| 費用 | 有免費額度，超過付費 | 免費 |
| 本機記憶體 | 0 | 0 |
| 隱私 | 資料上傳到雲端 | 區網內傳輸 |
| 品質 | 更好（1024 維） | 良好（768 維） |
| 離線可用 | ❌ | 需 24GB 機器在線 |
| 設定複雜度 | 簡單 | 需要設定遠端連線 |

---

## 第十部分：啟動與驗證

### 啟動 CoPaw 服務

```bash
# 確認 Ollama 正在運行（如果使用本地模型）
brew services list | grep ollama

# 前景啟動（開發/除錯用）
copaw app

# 背景啟動
nohup copaw app > ~/copaw.log 2>&1 &

# 檢查運行狀態
curl http://127.0.0.1:8088/api/version
```

### 完整驗證清單

| # | 驗證項目 | 方式 |
|---|---------|------|
| 1 | Web Console | 瀏覽器開啟 `http://127.0.0.1:8088` |
| 2 | AI 引擎 | Console 中輸入「你好，請自我介紹」 |
| 3 | Discord Bot | Discord 私訊 Bot「你好」 |
| 4 | Gmail 讀取 | Discord 告訴 Bot「請檢查我的 Gmail 收件箱」 |
| 5 | Gmail 發送 | Discord 告訴 Bot「發一封測試郵件到我自己」 |
| 6 | GitHub 操作 | Discord 告訴 Bot「建立 Hello World 專案並上傳 GitHub」 |
| 7 | Cron 任務 | `copaw cron list` 後 `copaw cron run <job_id>` |
| 8 | 記憶搜尋 | Console 中問「你還記得我上次說了什麼嗎？」 |
| 9 | 記憶體狀態 | 執行 `~/check-memory.sh` 確認無大量 Swap |
| 10 | 遠端 Ollama | `curl http://24GB機器IP:11434/api/tags`（如有設定） |

---

## 第十一部分：進階設定與維運

### 使用 launchd 開機自動啟動（推薦）

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
        <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/Users/你的使用者名稱/copaw-env/bin</string>
        <key>OLLAMA_NUM_PARALLEL</key>
        <string>1</string>
        <key>OLLAMA_KEEP_ALIVE</key>
        <string>10m</string>
        <key>OLLAMA_MAX_LOADED_MODELS</key>
        <string>1</string>
    </dict>
</dict>
</plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.copaw.agent.plist
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

### Q: 8B 模型載入後系統變慢

**原因**：8B 模型佔用約 5GB，加上系統和其他應用，記憶體可能不足。

**解法**：
```bash
# 關閉瀏覽器等大型應用
# 或使用更小的模型
ollama pull qwen3:4b

# 或改用 MoE 模型（只佔 2GB）
ollama pull qwen3:30b-a3b

# 或使用遠端 Ollama，完全不佔本機記憶體
copaw models set-llm  # 選 ollama-remote
```

### Q: 14B 模型載入失敗或極慢

**原因**：14B 需要約 8.5GB，16GB 機器記憶體不足。

**解法**：
- 改用 8B 模型（本地）
- 或使用遠端 Ollama 連接 24GB 機器（方案 C）
- 或使用 Claude API（方案 B）

### Q: 遠端 Ollama 連線失敗

**檢查清單**：
1. 24GB 機器 Ollama 是否運行：`curl http://24GB機器IP:11434/api/tags`
2. OLLAMA_HOST 是否已設定為 `0.0.0.0:11434`
3. 防火牆是否放行 11434 埠
4. base_url 格式需包含 `/v1`：`http://192.168.1.100:11434/v1`
5. IP 是否變動（建議設定 DHCP 保留）

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

### Q: Claude API 連線失敗

**替代方案**：使用 LiteLLM 作為 OpenAI 相容 Proxy：

```bash
pip install litellm
litellm --model claude-sonnet-4-20250514 --port 4000 --api_key sk-ant-你的Key &
```

然後在 CoPaw 中建立自訂 Provider，`base_url` 指向 `http://localhost:4000/v1`。

---

## 完整設定檔參考

### `~/.copaw/config.json`（16GB 版）

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

### `~/.copaw/providers.json`（16GB 三引擎：本地 + 遠端 + Claude）

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
    "provider_id": "ollama",
    "model": "qwen3:8b"
  }
}
```

### `~/.zshrc` Ollama 16GB 環境變數

```bash
# CoPaw + Ollama 16GB 保守設定
export OLLAMA_NUM_PARALLEL=1
export OLLAMA_KEEP_ALIVE=10m
export OLLAMA_MAX_LOADED_MODELS=1
export OLLAMA_GPU_LAYERS=999
```

### `~/.copaw/PROFILE.md`（16GB 版）

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
- 本地模型：Ollama qwen3:8b（日常對話）
- 遠端模型：遠端 Ollama qwen3:14b（需 24GB 機器在線）
- 雲端模型：Claude Sonnet 4（複雜任務）
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
- 模型選擇：簡單任務用本地 8B，複雜任務用遠端 14B 或 Claude
```

### 其他設定檔

`~/.config/himalaya/config.toml`、`~/.copaw/HEARTBEAT.md` 的設定與 [24GB 版指南](./mac-mini-m4-24g-copaw-implement.md#完整設定檔參考)相同。
