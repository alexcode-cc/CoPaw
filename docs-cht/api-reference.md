# API 參考文件

後端所有 API 以 **FastAPI** 實作，預設運行於 `http://127.0.0.1:8088`。所有 API 端點皆以 `/api` 為前綴。

> **提示**：若 `COPAW_OPENAPI_DOCS` 環境變數為 `true`，可在 `http://127.0.0.1:8088/docs` 查看互動式 Swagger 文件。

---

## 目錄

- [全域端點](#全域端點)
- [LLM Provider 管理（`/api/models`）](#llm-provider-管理apimodels)
- [設定管理（`/api/config`）](#設定管理apiconfig)
- [聊天管理（`/api/chats`）](#聊天管理apichats)
- [Agent 檔案與設定（`/api/agent`）](#agent-檔案與設定apiagent)
- [定時任務（`/api/cron`）](#定時任務apicron)
- [Skills 管理（`/api/skills`）](#skills-管理apiskills)
- [MCP 客戶端管理（`/api/mcp`）](#mcp-客戶端管理apimcp)
- [環境變數管理（`/api/envs`）](#環境變數管理apienvs)
- [工作目錄（`/api/workspace`）](#工作目錄apiworkspace)
- [本地模型管理（`/api/local-models`）](#本地模型管理apilocal-models)
- [Ollama 模型管理（`/api/ollama-models`）](#ollama-模型管理apiollama-models)
- [Console 推播（`/api/console`）](#console-推播apiconsole)
- [Agent 互動（`/api/agent/process`）](#agent-互動apiagentprocess)
- [通用錯誤格式與 CORS](#通用錯誤格式與-cors)

---

## 全域端點

| Method | 路徑 | 說明 |
|--------|------|------|
| `GET` | `/` | 回傳 Console SPA 的 `index.html` |
| `GET` | `/api/version` | 取得 CoPaw 版本號 |

---

## LLM Provider 管理（`/api/models`）

管理 LLM 服務提供商（OpenAI、DashScope、自訂 Provider 等）及其下的模型。

| Method | 路徑 | 說明 | 回傳 |
|--------|------|------|------|
| `GET` | `/api/models` | 列出所有 Provider | `List[ProviderInfo]` |
| `PUT` | `/api/models/{provider_id}/config` | 更新 Provider 設定（API Key / Base URL） | `ProviderInfo` |
| `POST` | `/api/models/custom-providers` | 建立自訂 Provider | `ProviderInfo` |
| `DELETE` | `/api/models/custom-providers/{provider_id}` | 刪除自訂 Provider | `List[ProviderInfo]` |
| `POST` | `/api/models/{provider_id}/test` | 測試 Provider 連線 | `TestConnectionResponse` |
| `POST` | `/api/models/{provider_id}/models/test` | 測試特定模型 | `TestConnectionResponse` |
| `POST` | `/api/models/{provider_id}/models` | 新增模型到 Provider | `ProviderInfo` |
| `DELETE` | `/api/models/{provider_id}/models/{model_id}` | 移除模型 | `ProviderInfo` |
| `GET` | `/api/models/active` | 取得目前啟用的 LLM | `ActiveModelsInfo` |
| `PUT` | `/api/models/active` | 設定啟用的 LLM | `ActiveModelsInfo` |

### ActiveModelsInfo 資料結構

```json
{
  "active_llm": {
    "provider_id": "dashscope",
    "model": "qwen-max"
  }
}
```

### 範例：設定啟用的 LLM

```http
PUT /api/models/active
Content-Type: application/json

{
  "active_llm": {
    "provider_id": "dashscope",
    "model": "qwen-max"
  }
}
```

---

## 設定管理（`/api/config`）

管理系統設定，包含各頻道設定與心跳設定。

| Method | 路徑 | 說明 | 回傳 |
|--------|------|------|------|
| `GET` | `/api/config/channels` | 列出所有頻道設定 | `dict` |
| `GET` | `/api/config/channels/types` | 列出可用頻道類型 | `List[str]` |
| `PUT` | `/api/config/channels` | 更新所有頻道設定 | `ChannelConfig` |
| `GET` | `/api/config/channels/{channel_name}` | 取得特定頻道設定 | `ChannelConfigUnion` |
| `PUT` | `/api/config/channels/{channel_name}` | 更新特定頻道設定 | `ChannelConfigUnion` |
| `GET` | `/api/config/heartbeat` | 取得心跳設定 | `dict` |
| `PUT` | `/api/config/heartbeat` | 更新心跳設定（支援熱重載） | `dict` |

### 範例：更新心跳設定

```http
PUT /api/config/heartbeat
Content-Type: application/json

{
  "enabled": true,
  "every": "6h",
  "target": "main",
  "activeHours": {
    "start": "08:00",
    "end": "22:00"
  }
}
```

> **注意**：心跳設定中的 `active_hours` 使用 JSON alias `activeHours`。

---

## 聊天管理（`/api/chats`）

管理聊天 Session 的建立、查詢與刪除。

| Method | 路徑 | 說明 | 回傳 |
|--------|------|------|------|
| `GET` | `/api/chats` | 列出聊天（可選 `user_id`、`channel` 查詢參數） | `list[ChatSpec]` |
| `POST` | `/api/chats` | 建立新聊天 | `ChatSpec` |
| `POST` | `/api/chats/batch-delete` | 批次刪除聊天 | `{deleted}` |
| `GET` | `/api/chats/{chat_id}` | 取得聊天歷史訊息 | `ChatHistory` |
| `PUT` | `/api/chats/{chat_id}` | 更新聊天（名稱等） | `ChatSpec` |
| `DELETE` | `/api/chats/{chat_id}` | 刪除聊天 | `{deleted}` |

### ChatSpec 資料結構

```json
{
  "id": "uuid-string",
  "name": "新對話",
  "session_id": "console_user_001",
  "user_id": "user_001",
  "channel": "console",
  "created_at": "2026-03-03T09:00:00+08:00",
  "updated_at": "2026-03-03T15:30:00+08:00",
  "meta": {}
}
```

---

## Agent 檔案與設定（`/api/agent`）

管理 Agent 工作目錄中的 Markdown 檔案（AGENTS.md、SOUL.md 等）和 memory 目錄的日記檔案。

| Method | 路徑 | 說明 | 回傳 |
|--------|------|------|------|
| `GET` | `/api/agent/files` | 列出工作目錄的 Markdown 檔案 | `list[MdFileInfo]` |
| `GET` | `/api/agent/files/{md_name}` | 讀取工作目錄的 Markdown 檔案 | `MdFileContent` |
| `PUT` | `/api/agent/files/{md_name}` | 寫入工作目錄的 Markdown 檔案 | `{written: bool}` |
| `GET` | `/api/agent/memory` | 列出 memory 目錄的 Markdown 檔案 | `list[MdFileInfo]` |
| `GET` | `/api/agent/memory/{md_name}` | 讀取 memory 目錄的 Markdown 檔案 | `MdFileContent` |
| `PUT` | `/api/agent/memory/{md_name}` | 寫入 memory 目錄的 Markdown 檔案 | `{written: bool}` |
| `GET` | `/api/agent/running-config` | 取得 Agent 運行設定 | `AgentsRunningConfig` |
| `PUT` | `/api/agent/running-config` | 更新 Agent 運行設定 | `AgentsRunningConfig` |

### 範例：讀取 AGENTS.md

```http
GET /api/agent/files/AGENTS.md
```

### 範例：更新 Agent 運行設定

```http
PUT /api/agent/running-config
Content-Type: application/json

{
  "max_iters": 80,
  "max_input_length": 131072
}
```

---

## 定時任務（`/api/cron`）

管理 Cron Job 排程任務，基於 APScheduler 實作。

| Method | 路徑 | 說明 | 回傳 |
|--------|------|------|------|
| `GET` | `/api/cron/jobs` | 列出所有 Cron 任務 | `list[CronJobSpec]` |
| `POST` | `/api/cron/jobs` | 建立 Cron 任務 | `CronJobSpec` |
| `GET` | `/api/cron/jobs/{job_id}` | 取得特定 Cron 任務 | `CronJobView` |
| `PUT` | `/api/cron/jobs/{job_id}` | 替換 Cron 任務 | `CronJobSpec` |
| `DELETE` | `/api/cron/jobs/{job_id}` | 刪除 Cron 任務 | `{deleted}` |
| `POST` | `/api/cron/jobs/{job_id}/pause` | 暫停 Cron 任務 | `{paused}` |
| `POST` | `/api/cron/jobs/{job_id}/resume` | 恢復 Cron 任務 | `{resumed}` |
| `POST` | `/api/cron/jobs/{job_id}/run` | 立即執行 Cron 任務 | `{started}` |
| `GET` | `/api/cron/jobs/{job_id}/state` | 取得 Cron 任務狀態 | `CronJobState` |

### CronJobSpec 資料結構

```json
{
  "id": "uuid-string",
  "name": "每日新聞摘要",
  "enabled": true,
  "task_type": "agent",
  "text": "",
  "request": { "query": "幫我整理今天的科技新聞摘要" },
  "dispatch": {
    "channel": "console",
    "target": {
      "user_id": "user_001",
      "session_id": "daily_news"
    },
    "meta": {}
  },
  "schedule": {
    "cron": "0 9 * * 1-5",
    "timezone": "Asia/Taipei"
  },
  "runtime": {
    "max_concurrency": 1,
    "misfire_grace_seconds": 300
  }
}
```

### task_type 說明

| 值 | 說明 |
|----|------|
| `"text"` | 直接發送文字到頻道（不經過 Agent） |
| `"agent"` | 將 query 送給 Agent 處理，結果回傳到頻道 |

### CronJobState 資料結構

```json
{
  "next_run_at": "2026-03-04T09:00:00+08:00",
  "last_run_at": "2026-03-03T09:00:00+08:00",
  "last_status": "success",
  "last_error": null
}
```

---

## Skills 管理（`/api/skills`）

管理 Agent 的能力擴展模組（Skills）。

| Method | 路徑 | 說明 | 回傳 |
|--------|------|------|------|
| `GET` | `/api/skills` | 列出所有 Skills（含是否啟用） | `list[SkillSpec]` |
| `GET` | `/api/skills/available` | 列出已啟用的 Skills | `list[SkillSpec]` |
| `GET` | `/api/skills/hub/search` | 搜尋 Hub Skills（`q` 關鍵字、`limit` 數量） | `list[HubSkillSpec]` |
| `POST` | `/api/skills/hub/install` | 從 Hub 安裝 Skill | `{installed, name, enabled, source_url}` |
| `POST` | `/api/skills` | 建立自訂 Skill | `{created}` |
| `POST` | `/api/skills/batch-enable` | 批次啟用多個 Skills | - |
| `POST` | `/api/skills/batch-disable` | 批次停用多個 Skills | - |
| `POST` | `/api/skills/{skill_name}/enable` | 啟用特定 Skill | `{enabled}` |
| `POST` | `/api/skills/{skill_name}/disable` | 停用特定 Skill | `{disabled}` |
| `DELETE` | `/api/skills/{skill_name}` | 刪除自訂 Skill（不可刪除內建 Skill） | `{deleted}` |
| `GET` | `/api/skills/{skill_name}/files/{source}/{file_path}` | 讀取 Skill 檔案內容 | `{content}` |

---

## MCP 客戶端管理（`/api/mcp`）

管理 Model Context Protocol（MCP）客戶端，用於整合外部工具服務。

| Method | 路徑 | 說明 | 回傳 |
|--------|------|------|------|
| `GET` | `/api/mcp` | 列出所有 MCP 客戶端 | `List[MCPClientInfo]` |
| `POST` | `/api/mcp` | 建立 MCP 客戶端 | `MCPClientInfo` |
| `GET` | `/api/mcp/{client_key}` | 取得特定 MCP 客戶端 | `MCPClientInfo` |
| `PUT` | `/api/mcp/{client_key}` | 更新 MCP 客戶端設定 | `MCPClientInfo` |
| `PATCH` | `/api/mcp/{client_key}/toggle` | 切換 MCP 客戶端啟用狀態 | `MCPClientInfo` |
| `DELETE` | `/api/mcp/{client_key}` | 刪除 MCP 客戶端 | `{message}` |

### MCPClientConfig 資料結構

```json
{
  "name": "tavily_search",
  "description": "",
  "transport": "stdio",
  "command": "uvx",
  "args": ["tavily-mcp@0.1.4"],
  "env": { "TAVILY_API_KEY": "tvly-xxx" },
  "cwd": "",
  "enabled": true,
  "url": "",
  "headers": {}
}
```

> **注意**：`MCPConfig.clients` 在 Pydantic 模型中是 `Dict[str, MCPClientConfig]`（以 client key 為 key 的字典），而非 List。

### transport 類型說明

| 類型 | 說明 | 使用時機 |
|------|------|---------|
| `stdio` | 啟動本地子進程，透過標準輸入/輸出通訊 | 本地 MCP Server（最常見） |
| `sse` | Server-Sent Events | 遠端 MCP Server（舊版協議） |
| `streamable_http` | HTTP Streaming | 遠端 MCP Server（新標準） |

---

## 環境變數管理（`/api/envs`）

管理儲存於工作目錄中的環境變數（`.env` 檔）。

| Method | 路徑 | 說明 | 回傳 |
|--------|------|------|------|
| `GET` | `/api/envs` | 列出所有環境變數 | `List[EnvVar]` |
| `PUT` | `/api/envs` | 批次儲存環境變數 | `List[EnvVar]` |
| `DELETE` | `/api/envs/{key}` | 刪除特定環境變數 | `List[EnvVar]` |

---

## 工作目錄（`/api/workspace`）

管理 `~/.copaw` 工作目錄的備份與還原。

| Method | 路徑 | 說明 | 回傳 |
|--------|------|------|------|
| `GET` | `/api/workspace/download` | 下載整個工作目錄（ZIP 格式） | `StreamingResponse` |
| `POST` | `/api/workspace/upload` | 上傳 ZIP 並合併到工作目錄 | `{success}` |

---

## 本地模型管理（`/api/local-models`）

管理本地下載的 LLM 模型（llama.cpp / MLX）。

| Method | 路徑 | 說明 | 回傳 |
|--------|------|------|------|
| `GET` | `/api/local-models` | 列出已下載的本地模型 | `List[LocalModelResponse]` |
| `POST` | `/api/local-models/download` | 背景下載模型 | `DownloadTaskResponse` |
| `GET` | `/api/local-models/download-status` | 取得下載任務狀態 | `List[DownloadTaskResponse]` |
| `DELETE` | `/api/local-models/{model_id}` | 刪除本地模型 | `{status, model_id}` |
| `POST` | `/api/local-models/cancel-download/{task_id}` | 取消下載任務 | `{status, task_id}` |

---

## Ollama 模型管理（`/api/ollama-models`）

管理 Ollama 服務的模型（需要先安裝 Ollama 服務）。

| Method | 路徑 | 說明 | 回傳 |
|--------|------|------|------|
| `GET` | `/api/ollama-models` | 列出 Ollama 模型 | `List[OllamaModelResponse]` |
| `POST` | `/api/ollama-models/download` | 背景下載 Ollama 模型 | `OllamaDownloadTaskResponse` |
| `GET` | `/api/ollama-models/download-status` | 取得下載任務狀態 | `List[OllamaDownloadTaskResponse]` |
| `DELETE` | `/api/ollama-models/download/{task_id}` | 取消下載任務 | `{status, task_id}` |
| `DELETE` | `/api/ollama-models/{name}` | 刪除 Ollama 模型 | `{status, name}` |

---

## Console 推播（`/api/console`）

用於 Cron 事件或背景任務完成後，向 Console 前端推播通知訊息。

| Method | 路徑 | 說明 | 回傳 |
|--------|------|------|------|
| `GET` | `/api/console/push-messages` | 取得待推播訊息 | `{messages}` |

**查詢參數：**
- `session_id`（可選）：只取得特定 Session 的推播訊息

**推播暫存機制：**
- 記憶體內暫存，上限 **500 則**訊息
- 訊息有效期限 **60 秒**
- `take()` 取出後自動移除

---

## Agent 互動（`/api/agent/process`）

透過 HTTP 向 Agent 發送查詢請求。此端點由 `agentscope_runtime.engine.app.AgentApp` 提供，掛載於 `/api/agent`。

> 此 API 主要由 Console Channel 前端使用，支援串流回應。

---

## 通用錯誤格式與 CORS

### 錯誤回傳格式

所有 API 回傳的錯誤格式統一為：

```json
{
  "detail": "錯誤訊息描述"
}
```

| HTTP 狀態碼 | 說明 |
|------------|------|
| `200` | 成功 |
| `400` | 請求參數錯誤 |
| `404` | 找不到資源 |
| `422` | 資料驗證失敗（Pydantic） |
| `500` | 伺服器內部錯誤 |

### CORS 設定

預設 CORS 允許所有來源（開發環境）。可透過環境變數限制：

```bash
export COPAW_CORS_ORIGINS="https://yourapp.com,http://localhost:3000"
```

---

## API 路由總覽

```
/                         → Console SPA (index.html)
/api/version              → 版本資訊
/api/models/*             → LLM Provider 管理
/api/config/*             → 頻道與心跳設定
/api/chats/*              → 聊天管理
/api/agent/files/*        → Agent 工作目錄 MD 檔
/api/agent/memory/*       → Agent 記憶日記 MD 檔
/api/agent/running-config → Agent 運行設定
/api/agent/process        → Agent 互動（AgentApp）
/api/cron/*               → 定時任務
/api/skills/*             → Skills 管理
/api/mcp/*                → MCP 客戶端管理
/api/envs/*               → 環境變數
/api/workspace/*          → 工作目錄備份還原
/api/local-models/*       → 本地模型管理
/api/ollama-models/*      → Ollama 模型管理
/api/console/*            → Console 推播
/assets/*                 → 靜態資源
/logo.png                 → Logo 圖示
/copaw-symbol.svg         → CoPaw 圖示
```
