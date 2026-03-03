# Mac Mini M4 (24GB) CoPaw 完整實作指南

> 本指南針對 **Mac Mini M4 (24GB)** 撰寫，相較於 16GB 版本，24GB 記憶體可運行更大參數的本地模型，支援並行推理與本地 Embedding，大幅提升離線 AI 能力。本文著重於 24GB 的模型選擇、記憶體優化與效能調校，其餘 Discord / Gmail / GitHub 設定與 [16GB 版指南](./mac-mini-m4-copaw-implement.md)相同。

---

## 目錄

- [16GB vs 24GB 差異總覽](#16gb-vs-24gb-差異總覽)
- [前置環境準備](#前置環境準備)
- [第一部分：安裝 CoPaw](#第一部分安裝-copaw)
- [第二部分：設定 AI 引擎](#第二部分設定-ai-引擎)
  - [方案 A：Ollama 本地模型（24GB 完整攻略）](#方案-aollama-本地模型24gb-完整攻略)
  - [方案 B：MLX 本地模型（Apple Silicon 原生）](#方案-bmlx-本地模型apple-silicon-原生)
  - [方案 C：Claude API（Anthropic）](#方案-cclaude-apianthropic)
  - [方案 D：混合架構（推薦）](#方案-d混合架構推薦)
- [第三部分：24GB 模型完整選擇指南](#第三部分24gb-模型完整選擇指南)
- [第四部分：記憶體效能優化](#第四部分記憶體效能優化)
- [第五部分：設定 Discord 機器人](#第五部分設定-discord-機器人)
- [第六部分：設定 Gmail 自動收發](#第六部分設定-gmail-自動收發)
- [第七部分：設定 GitHub 專案自動上傳](#第七部分設定-github-專案自動上傳)
- [第八部分：設定自動化工作流程](#第八部分設定自動化工作流程)
- [第九部分：本地 Embedding 與長期記憶](#第九部分本地-embedding-與長期記憶)
- [第十部分：啟動與驗證](#第十部分啟動與驗證)
- [第十一部分：進階設定與維運](#第十一部分進階設定與維運)
- [常見問題排除](#常見問題排除)
- [完整設定檔參考](#完整設定檔參考)

---

## 16GB vs 24GB 差異總覽

| 項目 | 16GB | 24GB |
|------|------|------|
| **可用模型範圍** | 7B ~ 14B | 7B ~ 32B（Q4 量化） |
| **推薦主力模型** | Qwen3 8B | Qwen3 14B / 32B（Q4） |
| **並行推理** | 不建議 | 可同時載入 2 個模型 |
| **本地 Embedding** | 不建議（用 API） | 可運行本地 Embedding 模型 |
| **MLX 後端** | 受限 | 完整支援，效能更優 |
| **Context 長度** | 128K（推理較慢） | 128K（舒適運行） |
| **CoPaw + 模型共存** | 記憶體緊張 | 充裕（CoPaw 約 500MB） |
| **Ollama 並行數** | 1 | 2 |

---

## 前置環境準備

### 硬體確認

Mac Mini M4 24GB 的統一記憶體架構讓 CPU 與 GPU 共享記憶體，搭配 Metal GPU 加速，是運行 14B ~ 32B 本地 LLM 的理想選擇。

### 記憶體規劃

```
24GB 總記憶體
├── macOS 系統          ~3-4 GB
├── CoPaw 服務          ~0.5 GB
├── 瀏覽器/其他應用       ~2-3 GB
├── 主力 LLM 模型       ~8-16 GB（依模型大小）
├── Embedding 模型       ~0.5-1 GB（可選）
└── 緩衝空間            ~2-4 GB
```

### 安裝基礎工具

```bash
# 安裝 Homebrew（如尚未安裝）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 安裝 Python 3.12（CoPaw 需要 3.10 ~ 3.13）
brew install python@3.12

# 安裝 Git
brew install git

# 安裝 GitHub CLI
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

### 方式二：pip 安裝（開發者推薦，含 MLX 支援）

```bash
# 建立虛擬環境
python3.12 -m venv ~/copaw-env
source ~/copaw-env/bin/activate

# 安裝 CoPaw（含 Ollama + MLX 支援）
pip install "copaw[ollama,mlx]"

# 安裝 Playwright 瀏覽器（browser_use 工具需要）
playwright install chromium
```

> **24GB 專屬**：多安裝了 `mlx` extra，讓你可以使用 Apple Silicon 原生的 MLX 後端運行模型。

### 初始化工作目錄

```bash
copaw init
```

互動式初始化引導：

1. **安全聲明確認**：CoPaw 的 Agent 可執行 Shell 指令，確認你了解風險
2. **選擇語言**：選 `zh`（中文）
3. **設定 LLM Provider**：先跳過，後面詳細設定
4. **選擇 Skills**：建議選 `all`（啟用所有內建 Skills）

> **快速初始化**：`copaw init --defaults`

---

## 第二部分：設定 AI 引擎

24GB 記憶體讓你有更多選擇。以下四種方案可以組合使用。

### 方案 A：Ollama 本地模型（24GB 完整攻略）

#### 步驟 A1：安裝與啟動 Ollama

```bash
brew install ollama

# 以背景服務啟動
brew services start ollama

# 確認運行
curl http://localhost:11434/api/tags
```

#### 步驟 A2：下載 24GB 適用模型

24GB 可以舒適運行 14B 模型，並可運行 32B Q4 量化模型。

```bash
# ═══════════════════════════════════════════
# 🏆 推薦首選：綜合能力最強
# ═══════════════════════════════════════════

# Qwen3 14B — 中英文兼優、推理能力強（~8.5GB 記憶體）
ollama pull qwen3:14b

# Qwen3 30B-A3B（MoE）— 30B 參數但僅啟用 3B（~2GB 記憶體，速度極快）
ollama pull qwen3:30b-a3b


# ═══════════════════════════════════════════
# 💻 程式開發專用
# ═══════════════════════════════════════════

# Qwen 2.5 Coder 14B — 程式撰寫能力頂尖（~8.5GB）
ollama pull qwen2.5-coder:14b

# Deepseek Coder V2 16B — 程式與推理兼具（~10GB）
ollama pull deepseek-coder-v2:16b

# Codestral 22B — Mistral 的程式專用模型（~13GB）
ollama pull codestral:22b


# ═══════════════════════════════════════════
# 🧠 極限挑戰：32B 模型（Q4 量化）
# ═══════════════════════════════════════════

# Qwen3 32B Q4_K_M — 最強本地模型（~18GB，接近極限）
ollama pull qwen3:32b

# Qwen 2.5 Coder 32B — 程式能力頂級（~18GB）
ollama pull qwen2.5-coder:32b

# ⚠ 32B 模型會佔用約 18GB，剩餘空間緊張
# 建議僅在不需要其他大型應用時使用


# ═══════════════════════════════════════════
# ⚡ 輕量快速（搭配 32B 主力使用）
# ═══════════════════════════════════════════

# Qwen3 8B — 快速回應、日常對話（~5GB）
ollama pull qwen3:8b

# Qwen3 4B — 極速、適合簡單任務（~2.5GB）
ollama pull qwen3:4b

# Llama 3.1 8B — 英文能力優秀（~5GB）
ollama pull llama3.1:8b
```

#### 步驟 A3：設定 Ollama 為 AI 引擎

```bash
copaw models set-llm
# 選 ollama → 選你下載的模型（推薦 qwen3:14b）
```

或編輯 `~/.copaw/providers.json`：

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
    "model": "qwen3:14b"
  }
}
```

#### 步驟 A4：Ollama 效能調校（24GB 專屬）

```bash
# ═══════════════════════════════════════════
# 24GB 專屬 Ollama 環境變數
# ═══════════════════════════════════════════

# 允許 2 個並行請求（24GB 有餘裕）
export OLLAMA_NUM_PARALLEL=2

# 模型保留在記憶體中 1 小時（減少重新載入）
export OLLAMA_KEEP_ALIVE=1h

# 限制模型最大使用記憶體為 18GB（保留 6GB 給系統）
export OLLAMA_MAX_LOADED_MODELS=2

# GPU 層數最大化（M4 Metal 加速）
export OLLAMA_GPU_LAYERS=999

# 加入 ~/.zshrc 使其永久生效
cat >> ~/.zshrc << 'ZSHRC'
export OLLAMA_NUM_PARALLEL=2
export OLLAMA_KEEP_ALIVE=1h
export OLLAMA_MAX_LOADED_MODELS=2
export OLLAMA_GPU_LAYERS=999
ZSHRC

source ~/.zshrc
```

**參數說明：**

| 環境變數 | 24GB 建議值 | 說明 |
|----------|------------|------|
| `OLLAMA_NUM_PARALLEL` | `2` | 同時處理 2 個請求（16GB 只建議 1） |
| `OLLAMA_KEEP_ALIVE` | `1h` | 模型保留 1 小時（省去重載時間） |
| `OLLAMA_MAX_LOADED_MODELS` | `2` | 最多同時載入 2 個模型（如主力 + Embedding） |
| `OLLAMA_GPU_LAYERS` | `999` | 所有層都用 GPU（M4 Metal 加速） |

---

### 方案 B：MLX 本地模型（Apple Silicon 原生）

MLX 是 Apple 專為 Apple Silicon 設計的機器學習框架，比 Ollama 的 llama.cpp 後端**更高效利用統一記憶體**。

#### 步驟 B1：安裝 MLX 依賴

如果安裝時已加上 `mlx` extra，則不需要額外安裝：

```bash
pip install "copaw[mlx]"
```

#### 步驟 B2：下載 MLX 模型

MLX 模型通常從 Hugging Face 下載，CoPaw 提供 CLI 命令：

```bash
# 下載 Qwen 2.5 14B MLX 量化版
copaw models download \
  "mlx-community/Qwen2.5-14B-Instruct-4bit" \
  -b mlx \
  -s huggingface

# 下載 Qwen 2.5 Coder 14B MLX 量化版
copaw models download \
  "mlx-community/Qwen2.5-Coder-14B-Instruct-4bit" \
  -b mlx \
  -s huggingface
```

確認下載完成：

```bash
copaw models local -b mlx
```

#### 步驟 B3：設定 MLX 為 AI 引擎

```bash
copaw models set-llm
# 選 mlx → 選擇下載的模型
```

#### MLX vs Ollama 效能對比

| 指標 | Ollama (llama.cpp) | MLX |
|------|-------------------|-----|
| Token/s（14B Q4） | ~25-35 | ~30-45 |
| 記憶體效率 | 良好 | 更好（原生統一記憶體） |
| 首次載入 | 較快 | 較慢（需編譯） |
| 模型格式 | GGUF（單檔） | safetensors（目錄） |
| 社群模型數量 | 極多 | 中等（mlx-community） |
| 量化選項 | Q2~Q8 多種 | 主要 4bit/8bit |

> **建議**：日常使用 Ollama（生態豐富），需要極致效能時切換 MLX。

---

### 方案 C：Claude API（Anthropic）

與 16GB 版設定完全相同。Claude API 不佔本地記憶體，可與本地模型並存。

#### 建立 Anthropic 自訂 Provider

```bash
# 建立 Provider
copaw models add-provider anthropic \
  -n "Anthropic (Claude)" \
  -u "https://api.anthropic.com/v1" \
  --api-key-prefix "sk-ant-"

# 設定 API Key
copaw models config-key anthropic

# 新增模型
copaw models add-model anthropic -m "claude-sonnet-4-20250514" -n "Claude Sonnet 4"
copaw models add-model anthropic -m "claude-opus-4-20250514" -n "Claude Opus 4"
copaw models add-model anthropic -m "claude-3-5-haiku-20241022" -n "Claude 3.5 Haiku"
```

或直接編輯 `~/.copaw/providers.json`（見[完整設定檔參考](#完整設定檔參考)）。

> 詳細步驟請參考 [16GB 版指南 — 方案 B](./mac-mini-m4-copaw-implement.md#方案-bclaude-apianthropic)。

---

### 方案 D：混合架構（推薦）

24GB 的最佳實踐是**混合使用本地模型與雲端 API**：

```
日常對話 / 簡單任務  →  Ollama Qwen3 14B（本地，免費）
程式開發 / 程式碼    →  Ollama Qwen2.5-Coder 14B（本地，免費）
複雜推理 / 長文撰寫  →  Claude Sonnet 4（雲端，按量付費）
Embedding / 記憶    →  Ollama nomic-embed-text（本地，免費）
```

**切換引擎：**

```bash
# 切到本地
copaw models set-llm  # 選 ollama → qwen3:14b

# 切到雲端
copaw models set-llm  # 選 anthropic → claude-sonnet-4-20250514
```

也可以透過 Discord 告訴 Agent：

```
你：接下來的任務比較複雜，請幫我切換到 Claude
```

### 作為遠端 Ollama 伺服器（供 16GB 本機使用）

若你同時擁有 **Mac Mini M4 16GB** 與 **24GB**，可將 24GB 機器設為 Ollama 伺服器，讓 16GB 本機的 CoPaw 透過網路呼叫遠端模型，使用 14B ~ 32B 等更大模型。

**24GB 機器需完成**：
1. 設定 `OLLAMA_HOST=0.0.0.0:11434` 以接受遠端連線
2. 防火牆放行 11434 埠
3. 下載 14B ~ 32B 模型

**完整遠端與本地設定步驟**請參考 [16GB 實作指南 — 方案 C：遠端 Ollama](./mac-mini-m4-copaw-implement.md#方案-c遠端-ollama24gb-機器)。

---

## 第三部分：24GB 模型完整選擇指南

### Ollama 模型推薦（按用途分類）

#### 綜合對話模型

| 模型 | 參數量 | 記憶體用量 | 推理速度 | 中文能力 | 推薦度 |
|------|--------|-----------|---------|---------|--------|
| `qwen3:32b` | 32B Q4 | ~18 GB | ~10 t/s | ★★★★★ | ⭐ 極限挑戰 |
| `qwen3:14b` | 14B | ~8.5 GB | ~30 t/s | ★★★★★ | ⭐⭐⭐ 最推薦 |
| `qwen3:30b-a3b` | 30B MoE | ~2 GB | ~60 t/s | ★★★★☆ | ⭐⭐ 極速 |
| `qwen3:8b` | 8B | ~5 GB | ~45 t/s | ★★★★☆ | ⭐⭐ 快速 |
| `llama3.1:8b` | 8B | ~5 GB | ~45 t/s | ★★☆☆☆ | 英文專用 |
| `gemma3:12b` | 12B | ~7.5 GB | ~35 t/s | ★★★☆☆ | 多模態 |
| `mistral:7b` | 7B | ~4.5 GB | ~50 t/s | ★★☆☆☆ | 輕量 |

#### 程式開發模型

| 模型 | 參數量 | 記憶體用量 | 推理速度 | 程式能力 | 推薦度 |
|------|--------|-----------|---------|---------|--------|
| `qwen2.5-coder:32b` | 32B Q4 | ~18 GB | ~10 t/s | ★★★★★ | ⭐ 極限 |
| `qwen2.5-coder:14b` | 14B | ~8.5 GB | ~30 t/s | ★★★★★ | ⭐⭐⭐ 最推薦 |
| `codestral:22b` | 22B | ~13 GB | ~18 t/s | ★★★★☆ | ⭐⭐ 強力 |
| `deepseek-coder-v2:16b` | 16B | ~10 GB | ~25 t/s | ★★★★☆ | ⭐⭐ |
| `qwen2.5-coder:7b` | 7B | ~4.5 GB | ~50 t/s | ★★★☆☆ | 快速 |

#### Embedding 模型（本地長期記憶用）

| 模型 | 維度 | 記憶體用量 | 說明 |
|------|------|-----------|------|
| `nomic-embed-text` | 768 | ~0.3 GB | 推薦，效能良好 |
| `mxbai-embed-large` | 1024 | ~0.7 GB | 更高品質 |
| `all-minilm` | 384 | ~0.1 GB | 極輕量 |

### 24GB 模型組合建議

#### 組合 1：全能組合（推薦）

```bash
ollama pull qwen3:14b           # 主力對話 (~8.5GB)
ollama pull qwen2.5-coder:14b   # 程式開發 (~8.5GB)
ollama pull nomic-embed-text    # 本地 Embedding (~0.3GB)
# 總計：不同時載入，按需切換
```

#### 組合 2：極速組合

```bash
ollama pull qwen3:30b-a3b       # MoE 極速對話 (~2GB)
ollama pull qwen3:8b            # 備用對話 (~5GB)
ollama pull qwen2.5-coder:7b    # 快速程式 (~4.5GB)
ollama pull nomic-embed-text    # Embedding (~0.3GB)
# 總計可同時載入：~12GB，記憶體非常充裕
```

#### 組合 3：極致品質

```bash
ollama pull qwen3:32b           # 最強對話 (~18GB)
ollama pull nomic-embed-text    # Embedding (~0.3GB)
# ⚠ 32B 載入後記憶體緊張，不建議同時運行其他大型應用
```

#### 組合 4：混合雲地

```bash
ollama pull qwen3:14b           # 本地日常 (~8.5GB)
ollama pull nomic-embed-text    # 本地 Embedding (~0.3GB)
# Claude Sonnet 4               # 雲端複雜任務（不佔記憶體）
```

---

## 第四部分：記憶體效能優化

### 4.1 macOS 記憶體壓力監控

```bash
# 即時監控記憶體
vm_stat 1

# 檢查 Swap 使用量（理想值為 0）
sysctl vm.swapusage

# 使用 Activity Monitor（圖形化）
open -a "Activity Monitor"
```

### 4.2 Ollama 記憶體管理策略

#### 策略一：按需載入（推薦）

讓 Ollama 自動管理模型的載入與卸載：

```bash
# 模型用完後 10 分鐘自動卸載（節省記憶體）
export OLLAMA_KEEP_ALIVE=10m

# 最多同時載入 1 個大模型
export OLLAMA_MAX_LOADED_MODELS=1
```

適合：同時運行其他應用（瀏覽器、IDE 等）

#### 策略二：常駐載入（低延遲）

讓主力模型常駐記憶體，消除首次載入延遲：

```bash
# 模型常駐 4 小時
export OLLAMA_KEEP_ALIVE=4h

# 允許 2 個模型同時載入（主力 + Embedding）
export OLLAMA_MAX_LOADED_MODELS=2
```

適合：CoPaw 專用機，不運行其他大型應用

#### 策略三：極致模式（24GB 全給 AI）

```bash
# 模型永不卸載
export OLLAMA_KEEP_ALIVE=-1

# 載入 2 個模型
export OLLAMA_MAX_LOADED_MODELS=2

# 2 個並行請求
export OLLAMA_NUM_PARALLEL=2
```

適合：Mac Mini 完全作為 AI 伺服器使用

### 4.3 macOS 系統層級優化

#### 關閉不必要的系統服務

```bash
# 減少 Spotlight 索引的記憶體佔用
sudo mdutil -a -i off

# 需要時再開啟
# sudo mdutil -a -i on
```

#### 最佳化 Swap 設定

macOS 預設使用 SSD 作為 Swap，M4 的 SSD 速度很快，但頻繁 Swap 仍會影響效能：

```bash
# 檢查目前 Swap 使用量
sysctl vm.swapusage

# 理想狀態：Swap 使用量為 0 或很小
# 如果 Swap 超過 2GB，考慮使用更小的模型
```

#### 登入項目精簡

「系統設定 → 一般 → 登入項目與延伸功能」中，移除不必要的開機啟動項目，釋放更多記憶體給 AI 模型。

### 4.4 CoPaw 層級優化

#### 增加 Context 長度（24GB 可承受更多）

```bash
# 編輯 config.json 或透過 API
curl -X PUT http://127.0.0.1:8088/api/agent/running-config \
  -H "Content-Type: application/json" \
  -d '{
    "max_iters": 80,
    "max_input_length": 131072
  }'
```

| 設定 | 16GB 建議 | 24GB 建議 | 說明 |
|------|----------|----------|------|
| `max_iters` | 50 | 80 | 更多迭代，處理更複雜任務 |
| `max_input_length` | 131072 | 131072 | 128K tokens（兩者皆可） |
| `MEMORY_COMPACT_RATIO` | 0.7 | 0.8 | 更晚觸發壓縮，保留更多對話歷史 |
| `MEMORY_COMPACT_KEEP_RECENT` | 3 | 5 | 壓縮後保留更多最近對話 |

```bash
# 調整記憶壓縮參數
copaw env set COPAW_MEMORY_COMPACT_RATIO "0.8"
copaw env set COPAW_MEMORY_COMPACT_KEEP_RECENT "5"
```

### 4.5 模型即時切換腳本

建立一個便捷腳本，在不同場景間切換模型：

```bash
cat > ~/switch-copaw-model.sh << 'SCRIPT'
#!/bin/bash
echo "CoPaw 模型切換"
echo "1) Qwen3 14B（綜合對話）"
echo "2) Qwen2.5-Coder 14B（程式開發）"
echo "3) Qwen3 32B（極致品質）"
echo "4) Qwen3 30B-A3B（極速 MoE）"
echo "5) Claude Sonnet 4（雲端）"
read -p "選擇 [1-5]: " choice

case $choice in
  1) MODEL="ollama:qwen3:14b" ;;
  2) MODEL="ollama:qwen2.5-coder:14b" ;;
  3) MODEL="ollama:qwen3:32b" ;;
  4) MODEL="ollama:qwen3:30b-a3b" ;;
  5) MODEL="anthropic:claude-sonnet-4-20250514" ;;
  *) echo "無效選擇"; exit 1 ;;
esac

PROVIDER=$(echo $MODEL | cut -d: -f1)
MODEL_NAME=$(echo $MODEL | cut -d: -f2-)

curl -s -X PUT http://127.0.0.1:8088/api/models/active \
  -H "Content-Type: application/json" \
  -d "{\"active_llm\":{\"provider_id\":\"$PROVIDER\",\"model\":\"$MODEL_NAME\"}}"

echo "已切換到 $MODEL"
SCRIPT

chmod +x ~/switch-copaw-model.sh
```

---

## 第五部分：設定 Discord 機器人

與 16GB 版完全相同，請參考 [16GB 版指南 — 第三部分](./mac-mini-m4-copaw-implement.md#第三部分設定-discord-機器人)。

**簡要步驟：**

1. 在 [Discord Developer Portal](https://discord.com/developers/applications) 建立 Bot
2. 開啟 **Message Content Intent**
3. 取得 Bot Token
4. 設定 CoPaw：

```bash
copaw channels config
# 選 discord → 輸入 Bot Token
```

或編輯 `~/.copaw/config.json`：

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "bot_token": "你的Discord Bot Token"
    },
    "console": { "enabled": true }
  }
}
```

---

## 第六部分：設定 Gmail 自動收發

與 16GB 版完全相同，請參考 [16GB 版指南 — 第四部分](./mac-mini-m4-copaw-implement.md#第四部分設定-gmail-自動收發)。

**簡要步驟：**

1. `brew install himalaya`
2. 取得 Gmail App Password（需啟用兩步驟驗證）
3. 儲存密碼到 macOS 鑰匙圈
4. 建立 `~/.config/himalaya/config.toml`（Gmail IMAP/SMTP 設定）
5. 建立自訂 himalaya Skill（覆蓋內建版本以啟用發送功能）
6. `copaw skills enable himalaya`

> **重要**：內建 himalaya Skill 預設**停用郵件發送**，必須建立自訂版本。詳見 16GB 版指南。

---

## 第七部分：設定 GitHub 專案自動上傳

與 16GB 版完全相同，請參考 [16GB 版指南 — 第五部分](./mac-mini-m4-copaw-implement.md#第五部分設定-github-專案自動上傳)。

**簡要步驟：**

```bash
git config --global user.name "你的名稱"
git config --global user.email "你的Gmail@gmail.com"
gh auth login
ssh-keygen -t ed25519 -C "你的Gmail@gmail.com"
gh ssh-key add ~/.ssh/id_ed25519.pub --title "Mac Mini M4 24GB"
```

然後編輯 `~/.copaw/PROFILE.md` 加入 GitHub 資訊。

---

## 第八部分：設定自動化工作流程

與 16GB 版相同的四個工作流程：

### 工作流程 1：定時檢查 Gmail

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

### 工作流程 2：Discord 建立專案並上傳 GitHub

直接在 Discord 對 Bot 說：

```
幫我建立一個 Python CLI 專案叫做 "hello-world"，
功能是簡單的待辦事項管理工具，完成後上傳到我的 GitHub。
```

### 工作流程 3：每日工作彙報（Discord + Gmail 通知）

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

編輯 `~/.copaw/HEARTBEAT.md` 並啟用（每 2 小時，08:00-22:00）。

> 完整 Heartbeat 設定請見 [16GB 版指南 — 第六部分](./mac-mini-m4-copaw-implement.md#第六部分設定自動化工作流程)。

---

## 第九部分：本地 Embedding 與長期記憶

> **24GB 獨有優勢**：16GB 版建議使用雲端 Embedding API，但 24GB 可以運行本地 Embedding 模型，實現完全離線的長期記憶功能。

### 方案 1：Ollama 本地 Embedding（推薦）

```bash
# 下載 Embedding 模型
ollama pull nomic-embed-text

# 驗證模型已載入
ollama list
```

設定 CoPaw 使用本地 Embedding：

```bash
# Ollama 提供 OpenAI 相容的 Embedding API
copaw env set EMBEDDING_BASE_URL "http://localhost:11434/v1"
copaw env set EMBEDDING_API_KEY "ollama"
copaw env set EMBEDDING_MODEL_NAME "nomic-embed-text"
copaw env set EMBEDDING_DIMENSIONS "768"

# 啟用記憶管理
copaw env set ENABLE_MEMORY_MANAGER "true"
copaw env set MEMORY_STORE_BACKEND "chroma"
```

### 方案 2：DashScope 雲端 Embedding

如果不想用本地資源：

```bash
copaw env set EMBEDDING_API_KEY "sk-你的DashScope-Key"
copaw env set EMBEDDING_MODEL_NAME "text-embedding-v4"
copaw env set EMBEDDING_DIMENSIONS "1024"
copaw env set ENABLE_MEMORY_MANAGER "true"
```

### 本地 vs 雲端 Embedding 對比

| 指標 | 本地（nomic-embed-text） | 雲端（DashScope） |
|------|------------------------|-------------------|
| 費用 | 免費 | 有免費額度，超過付費 |
| 記憶體 | ~300MB | 0 |
| 隱私 | 完全離線 | 資料上傳到雲端 |
| 速度 | 較快（本地） | 取決於網路 |
| 品質 | 良好 | 更好（1024 維） |
| 離線可用 | ✅ | ❌ |

---

## 第十部分：啟動與驗證

### 啟動服務

```bash
# 確認 Ollama 正在運行
brew services list | grep ollama

# 啟動 CoPaw
copaw app

# 或背景啟動
nohup copaw app > ~/copaw.log 2>&1 &
```

### 完整驗證清單

| # | 驗證項目 | 方式 |
|---|---------|------|
| 1 | Web Console | 瀏覽器開啟 `http://127.0.0.1:8088` |
| 2 | AI 引擎 | Console 中輸入「你好，請自我介紹」 |
| 3 | Discord Bot | Discord 私訊 Bot「你好」 |
| 4 | Gmail 讀取 | Discord 告訴 Bot「請檢查我的 Gmail 收件箱」 |
| 5 | Gmail 發送 | Discord 告訴 Bot「發一封測試郵件到我自己」 |
| 6 | GitHub | Discord 告訴 Bot「建立 Hello World 專案並上傳 GitHub」 |
| 7 | Cron 任務 | `copaw cron list` 後 `copaw cron run <job_id>` |
| 8 | 記憶搜尋 | Console 中問「你還記得我上次說了什麼嗎？」 |
| 9 | 記憶體狀態 | `vm_stat` 確認無大量 Swap |

---

## 第十一部分：進階設定與維運

### launchd 開機自動啟動（推薦）

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
        <key>OLLAMA_NUM_PARALLEL</key>
        <string>2</string>
        <key>OLLAMA_KEEP_ALIVE</key>
        <string>1h</string>
        <key>OLLAMA_MAX_LOADED_MODELS</key>
        <string>2</string>
    </dict>
</dict>
</plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.copaw.agent.plist
```

### 備份與還原

```bash
# 備份
curl -o copaw-backup.zip http://127.0.0.1:8088/api/workspace/download

# 還原
curl -X POST http://127.0.0.1:8088/api/workspace/upload \
  -F "file=@copaw-backup.zip"
```

### 記憶體監控腳本

建立一個定期監控腳本，確保記憶體壓力在合理範圍：

```bash
cat > ~/check-memory.sh << 'SCRIPT'
#!/bin/bash
echo "=== Mac Mini M4 24GB 記憶體狀態 ==="
echo ""

# 系統記憶體
echo "--- 系統記憶體 ---"
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

SCRIPT

chmod +x ~/check-memory.sh
```

---

## 常見問題排除

### Q: 32B 模型載入後系統變慢

**原因**：32B Q4 模型佔用約 18GB，剩餘記憶體不足。

**解法**：
```bash
# 切換到 14B 模型
copaw models set-llm  # 選 ollama → qwen3:14b

# 或使用 MoE 模型（只佔 2GB）
copaw models set-llm  # 選 ollama → qwen3:30b-a3b

# 強制卸載所有 Ollama 模型
curl -X DELETE http://localhost:11434/api/generate -d '{"model":"qwen3:32b","keep_alive":0}'
```

### Q: 兩個模型同時載入時記憶體不足

**解法**：

```bash
# 限制為 1 個同時載入
export OLLAMA_MAX_LOADED_MODELS=1

# 縮短保留時間
export OLLAMA_KEEP_ALIVE=5m
```

### Q: MLX 模型載入很慢

**原因**：MLX 首次載入需要編譯計算圖，後續會有快取。

**解法**：第一次載入耐心等待（通常 30-60 秒），後續載入會快很多。

### Q: 本地 Embedding 搜尋結果品質差

**解法**：

```bash
# 改用更高品質的 Embedding 模型
ollama pull mxbai-embed-large

copaw env set EMBEDDING_MODEL_NAME "mxbai-embed-large"
copaw env set EMBEDDING_DIMENSIONS "1024"
```

### Q: Discord Bot / Gmail / GitHub 相關問題

請參考 [16GB 版指南 — 常見問題排除](./mac-mini-m4-copaw-implement.md#常見問題排除)。

---

## 完整設定檔參考

### `~/.copaw/config.json` 完整範例（24GB 優化版）

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
    "console": { "enabled": true },
    "telegram": { "enabled": false },
    "dingtalk": { "enabled": false },
    "feishu": { "enabled": false },
    "qq": { "enabled": false },
    "imessage": { "enabled": false }
  },
  "mcp": {
    "clients": {}
  },
  "agents": {
    "language": "zh",
    "running": {
      "max_iters": 80,
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

### `~/.copaw/providers.json` 完整範例（24GB 三引擎）

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
    "provider_id": "ollama",
    "model": "qwen3:14b"
  }
}
```

### `~/.zshrc` Ollama 24GB 環境變數

```bash
# CoPaw + Ollama 24GB 優化設定
export OLLAMA_NUM_PARALLEL=2
export OLLAMA_KEEP_ALIVE=1h
export OLLAMA_MAX_LOADED_MODELS=2
export OLLAMA_GPU_LAYERS=999
```

### `~/.copaw/PROFILE.md` 完整範例（24GB 版）

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

- 設備：Mac Mini M4 24GB
- 本地模型：Ollama qwen3:14b（主力）、qwen2.5-coder:14b（程式）
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
- 模型選擇：日常用本地 Qwen3 14B，複雜任務切換 Claude
```

### 其他設定檔

`~/.config/himalaya/config.toml`、`~/.copaw/HEARTBEAT.md` 與 16GB 版完全相同，請參考 [16GB 版指南 — 完整設定檔參考](./mac-mini-m4-copaw-implement.md#完整設定檔參考)。
