# 設定系統文件

CoPaw 的設定系統基於**檔案系統**，所有設定以 JSON/Markdown 格式儲存於工作目錄（預設 `~/.copaw/`）。本文件說明所有可用的設定項目與環境變數。

---

## 目錄

- [工作目錄（WORKING_DIR）](#工作目錄working_dir)
- [主設定（config.json）](#主設定configjson)
- [頻道設定](#頻道設定)
- [MCP 客戶端設定](#mcp-客戶端設定)
- [Agent 行為設定](#agent-行為設定)
- [心跳設定](#心跳設定)
- [LLM Provider 設定（providers.json）](#llm-provider-設定providersjson)
- [環境變數參考](#環境變數參考)
- [Pydantic 設定模型](#pydantic-設定模型)

---

## 工作目錄（WORKING_DIR）

工作目錄是 CoPaw 儲存所有資料的根目錄。

| 項目 | 說明 |
|------|------|
| **預設路徑** | `~/.copaw/`（使用者家目錄下的 `.copaw`） |
| **自訂方式** | 設定環境變數 `COPAW_WORKING_DIR` |

```
~/.copaw/
├── config.json          # 主要設定
├── providers.json       # LLM Provider 設定
├── jobs.json            # Cron Job 清單
├── chats.json           # 聊天對話清單
│
├── AGENTS.md            # Agent 工作流程與規則（構成 System Prompt）
├── SOUL.md              # Agent 核心身份與行為原則
├── PROFILE.md           # Agent 與使用者個人資料（可選）
├── MEMORY.md            # 長期記憶（人工或 AI 更新）
├── HEARTBEAT.md         # 心跳任務指令清單
│
├── active_skills/       # 當前啟用的 Skills（Agent 載入）
├── customized_skills/   # 使用者自訂 Skills
├── memory/              # 每日日記（YYYY-MM-DD.md）
├── sessions/            # 會話狀態（JSON 格式）
├── models/              # 本地下載的 LLM 模型
│   └── manifest.json    # 本地模型清單
└── custom_channels/     # 自訂頻道
```

---

## 主設定（`config.json`）

`config.json` 是系統主設定檔，對應 Pydantic `Config` 模型。可透過 API 或直接編輯檔案修改。

**完整結構範例：**

```json
{
  "channels": {
    "console": { "enabled": true },
    "telegram": {
      "bot_token": "",
      "http_proxy": "",
      "show_typing": true
    },
    "discord": {
      "bot_token": "",
      "http_proxy": ""
    },
    "dingtalk": {
      "client_id": "",
      "client_secret": "",
      "media_dir": ""
    },
    "feishu": {
      "app_id": "",
      "app_secret": "",
      "encrypt_key": "",
      "verification_token": "",
      "media_dir": ""
    },
    "qq": {
      "app_id": "",
      "client_secret": ""
    },
    "imessage": {
      "db_path": "",
      "poll_sec": 5
    }
  },
  "mcp": {
    "clients": [
      {
        "name": "tavily_search",
        "transport": "stdio",
        "command": "uvx",
        "args": ["tavily-mcp@0.1.4"],
        "env": { "TAVILY_API_KEY": "" },
        "cwd": "",
        "enabled": false,
        "url": "",
        "headers": {}
      }
    ]
  },
  "last_api": {
    "host": "127.0.0.1",
    "port": 8088
  },
  "agents": {
    "language": "zh",
    "running": {
      "max_iters": 50,
      "max_input_length": 131072
    },
    "defaults": {
      "heartbeat": {
        "enabled": false,
        "every": "6h",
        "target": "main",
        "active_hours": null
      }
    }
  },
  "show_tool_details": true
}
```

---

## 頻道設定

所有頻道繼承自 `BaseChannelConfig`，共有欄位：

| 欄位 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `enabled` | bool | `False`（Console 除外） | 是否啟用此頻道 |
| `bot_prefix` | str | `""` | Bot 指令前綴 |

> **注意**：`ChannelConfig` 設定 `model_config = ConfigDict(extra="allow")`，允許額外欄位。

### Console 頻道（Web UI）

Console 頻道**預設啟用**（`enabled=True`），提供網頁版聊天介面。

```json
{
  "console": {
    "enabled": true
  }
}
```

### Telegram 頻道

| 欄位 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `bot_token` | str | `""` | Telegram Bot Token（從 @BotFather 取得） |
| `http_proxy` | str | `""` | 代理伺服器（如 `socks5://127.0.0.1:7890`） |
| `http_proxy_auth` | str | `""` | 代理認證（`username:password`） |
| `show_typing` | Optional[bool] | `None` | 是否顯示「輸入中...」狀態 |

### Discord 頻道

| 欄位 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `bot_token` | str | `""` | Discord Bot Token |
| `http_proxy` | str | `""` | 代理伺服器 |
| `http_proxy_auth` | str | `""` | 代理認證 |

### DingTalk（鈐釘釘）頻道

| 欄位 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `client_id` | str | `""` | DingTalk App Client ID |
| `client_secret` | str | `""` | DingTalk App Client Secret |
| `media_dir` | str | `"~/.copaw/media"` | 媒體檔案儲存目錄 |

### Feishu（飛書）頻道

| 欄位 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `app_id` | str | `""` | 飛書 App ID |
| `app_secret` | str | `""` | 飛書 App Secret |
| `encrypt_key` | str | `""` | 訊息加密 Key |
| `verification_token` | str | `""` | 訊息驗證 Token |
| `media_dir` | str | `"~/.copaw/media"` | 媒體檔案儲存目錄 |

### QQ 頻道

| 欄位 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `app_id` | str | `""` | QQ App ID |
| `client_secret` | str | `""` | QQ App Secret |

### iMessage 頻道（macOS 限定）

| 欄位 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `db_path` | str | `"~/Library/Messages/chat.db"` | iMessage SQLite DB 路徑 |
| `poll_sec` | float | `1.0` | 輪詢間隔秒數 |

---

## MCP 客戶端設定

MCP（Model Context Protocol）客戶端設定允許 Agent 使用外部工具服務。

### MCPConfig 結構

> **重要**：`MCPConfig.clients` 是 `Dict[str, MCPClientConfig]`（字典），以 client key 作為 key，不是 List。

### MCPClientConfig 欄位說明

| 欄位 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `name` | str | （必填） | 客戶端顯示名稱 |
| `description` | str | `""` | 客戶端描述 |
| `enabled` | bool | `True` | 是否啟用 |
| `transport` | `"stdio"` \| `"sse"` \| `"streamable_http"` | `"stdio"` | 傳輸協議 |
| `command` | str | `""` | 啟動命令（`stdio` 模式） |
| `args` | List[str] | `[]` | 命令參數 |
| `env` | Dict[str, str] | `{}` | 環境變數 |
| `cwd` | str | `""` | 工作目錄 |
| `url` | str | `""` | 服務 URL（`sse`/`streamable_http` 模式） |
| `headers` | Dict[str, str] | `{}` | HTTP 請求標頭 |

### 常用 MCP 客戶端範例

#### Tavily 網頁搜尋（預設內建）
```json
{
  "name": "tavily_search",
  "transport": "stdio",
  "command": "uvx",
  "args": ["tavily-mcp@0.1.4"],
  "env": { "TAVILY_API_KEY": "tvly-your-key" },
  "enabled": true
}
```

#### Playwright 瀏覽器操作
```json
{
  "name": "playwright",
  "transport": "stdio",
  "command": "npx",
  "args": ["@playwright/mcp@latest"],
  "env": {},
  "enabled": true
}
```

#### 遠端 SSE 服務
```json
{
  "name": "my_remote_service",
  "transport": "sse",
  "url": "https://my-mcp-server.com/sse",
  "headers": { "Authorization": "Bearer my-token" },
  "enabled": true
}
```

---

## Agent 行為設定

在 `config.json` 的 `agents` 區段設定。

### 語言設定

```json
{
  "agents": {
    "language": "zh",
    "installed_md_files_language": null
  }
}
```

| 欄位 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `language` | str | `"zh"` | Agent 語言（`"zh"` 或 `"en"`） |
| `installed_md_files_language` | Optional[str] | `null` | 已安裝的 MD 範本語言（自動追蹤） |

| language 值 | 說明 |
|-------------|------|
| `"zh"` | 中文（載入 `md_files/zh/` 下的 MD 範本） |
| `"en"` | 英文（載入 `md_files/en/` 下的 MD 範本） |

### 執行參數

```json
{
  "agents": {
    "running": {
      "max_iters": 50,
      "max_input_length": 131072
    }
  }
}
```

| 欄位 | 型別 | 預設值 | 限制 |
|------|------|--------|------|
| `max_iters` | int | `50` | `≥ 1` |
| `max_input_length` | int | `131072` | `≥ 1000` |

### 顯示設定

```json
{
  "show_tool_details": true
}
```

設為 `false` 時，前端不顯示工具呼叫的細節（輸入/輸出）。

---

## 心跳設定

心跳設定在 `config.json` 的 `agents.defaults.heartbeat` 區段，也可透過 API 單獨更新。

```json
{
  "heartbeat": {
    "enabled": true,
    "every": "6h",
    "target": "main",
    "active_hours": {
      "start": "08:00",
      "end": "22:00",
      "timezone": "Asia/Taipei"
    }
  }
}
```

| 欄位 | 說明 | 預設值 |
|------|------|--------|
| `enabled` | 是否啟用心跳 | `false` |
| `every` | 執行間隔（`30m`、`1h`、`6h`、`1d` 等） | `"6h"` |
| `target` | 結果發送至哪個 Session/頻道。設為 `"last"` 可自動 dispatch 到上次對話 | `"main"` |
| `active_hours`（alias: `activeHours`） | 只在此時段內執行（可選） | `null`（全天） |

**ActiveHoursConfig 欄位：**

| 欄位 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `start` | str | `"08:00"` | 活躍時段開始（HH:MM） |
| `end` | str | `"22:00"` | 活躍時段結束（HH:MM） |

> **心跳機制說明**：每次觸發時，Agent 會讀取 `~/.copaw/HEARTBEAT.md` 的內容並執行其中的任務清單（如：檢查郵件、整理行事曆、發送摘要等）。

---

## LLM Provider 設定（`providers.json`）

`providers.json` 儲存所有 LLM 服務提供商的設定。**注意：此檔案包含 API Key，請勿提交到版本控制。**

### 內建 Provider 範例

```json
{
  "providers": [
    {
      "id": "dashscope",
      "name": "DashScope（阿里雲）",
      "api_key": "sk-your-dashscope-key",
      "base_url": "https://dashscope.aliyuncs.com/compatible-mode/v1",
      "models": [
        { "id": "qwen-max", "name": "Qwen Max", "enabled": true },
        { "id": "qwen-plus", "name": "Qwen Plus", "enabled": true },
        { "id": "qwen-turbo", "name": "Qwen Turbo", "enabled": true }
      ]
    },
    {
      "id": "openai",
      "name": "OpenAI",
      "api_key": "sk-your-openai-key",
      "base_url": "https://api.openai.com/v1",
      "models": [
        { "id": "gpt-4o", "name": "GPT-4o", "enabled": true }
      ]
    }
  ],
  "active": {
    "provider_id": "dashscope",
    "model_id": "qwen-max"
  }
}
```

---

## 環境變數參考

環境變數可透過系統設定或工作目錄的 `.env` 檔案（`/envs` API 管理）設定。

### 系統級環境變數

| 環境變數 | 說明 | 預設值 |
|----------|------|--------|
| `COPAW_WORKING_DIR` | 工作目錄路徑 | `~/.copaw` |
| `COPAW_JOBS_FILE` | Cron Jobs 檔案名稱 | `jobs.json` |
| `COPAW_CHATS_FILE` | 聊天記錄檔案名稱 | `chats.json` |
| `COPAW_CONFIG_FILE` | 設定檔名稱 | `config.json` |
| `COPAW_HEARTBEAT_FILE` | Heartbeat MD 檔案名稱 | `HEARTBEAT.md` |
| `COPAW_LOG_LEVEL` | 日誌等級（`DEBUG`/`INFO`/`WARNING`/`ERROR`） | `INFO` |
| `COPAW_RUNNING_IN_CONTAINER` | 是否在容器中執行（`true`/`false`） | `false` |
| `COPAW_CORS_ORIGINS` | CORS 允許來源（逗號分隔） | `*` |
| `COPAW_OPENAPI_DOCS` | 是否開放 OpenAPI 文件 | `false` |

### 記憶體管理環境變數

| 環境變數 | 說明 | 預設值 |
|----------|------|--------|
| `ENABLE_MEMORY_MANAGER` | 是否啟用長期記憶管理 | `true` |
| `MEMORY_STORE_BACKEND` | 記憶向量後端（`auto`/`local`/`chroma`） | `auto` |
| `COPAW_MEMORY_COMPACT_KEEP_RECENT` | 記憶壓縮時保留最近 N 輪對話 | `3` |
| `COPAW_MEMORY_COMPACT_RATIO` | 記憶壓縮觸發比例（Token 占比） | `0.7` |
| `EMBEDDING_API_KEY` | 向量嵌入 API Key | - |
| `EMBEDDING_BASE_URL` | 向量嵌入 Base URL | DashScope URL |
| `EMBEDDING_MODEL_NAME` | 嵌入模型名稱 | `text-embedding-v4` |
| `EMBEDDING_DIMENSIONS` | 嵌入向量維度 | `1024` |

> **Windows 說明**：Windows 系統因 Chroma 相容性問題，`MEMORY_STORE_BACKEND` 預設為 `local`。其他平台預設為 `chroma`。

### LLM 相關環境變數

| 環境變數 | 說明 |
|----------|------|
| `DASHSCOPE_API_KEY` | 阿里雲 DashScope API Key |
| `DASHSCOPE_BASE_URL` | DashScope Base URL |
| `TAVILY_API_KEY` | Tavily 網頁搜尋 API Key（MCP 客戶端使用） |

### 瀏覽器相關

| 環境變數 | 說明 |
|----------|------|
| `PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH` | 自訂 Chromium 瀏覽器路徑 |

---

## Pydantic 設定模型

以下為 `src/copaw/config/config.py` 中定義的完整 Pydantic 模型結構，供開發者擴展使用。

```python
class BaseChannelConfig(BaseModel):
    enabled: bool = False
    bot_prefix: str = ""

class ConsoleConfig(BaseChannelConfig):
    enabled: bool = True  # Console 預設啟用

class IMessageChannelConfig(BaseChannelConfig):
    db_path: str = "~/Library/Messages/chat.db"
    poll_sec: float = 1.0

class DiscordConfig(BaseChannelConfig):
    bot_token: str = ""
    http_proxy: str = ""
    http_proxy_auth: str = ""

class DingTalkConfig(BaseChannelConfig):
    client_id: str = ""
    client_secret: str = ""
    media_dir: str = "~/.copaw/media"

class FeishuConfig(BaseChannelConfig):
    app_id: str = ""
    app_secret: str = ""
    encrypt_key: str = ""
    verification_token: str = ""
    media_dir: str = "~/.copaw/media"

class QQConfig(BaseChannelConfig):
    app_id: str = ""
    client_secret: str = ""

class TelegramConfig(BaseChannelConfig):
    bot_token: str = ""
    http_proxy: str = ""
    http_proxy_auth: str = ""
    show_typing: Optional[bool] = None

class ChannelConfig(BaseModel):
    model_config = ConfigDict(extra="allow")  # 允許額外欄位
    imessage: IMessageChannelConfig
    discord: DiscordConfig
    dingtalk: DingTalkConfig
    feishu: FeishuConfig
    qq: QQConfig
    telegram: TelegramConfig
    console: ConsoleConfig

class LastApiConfig(BaseModel):
    host: Optional[str] = None
    port: Optional[int] = None

class ActiveHoursConfig(BaseModel):
    start: str = "08:00"
    end: str = "22:00"

class HeartbeatConfig(BaseModel):
    enabled: bool = False
    every: str = "6h"      # HEARTBEAT_DEFAULT_EVERY
    target: str = "main"   # HEARTBEAT_DEFAULT_TARGET
    active_hours: Optional[ActiveHoursConfig] = None  # alias: "activeHours"

class AgentsDefaultsConfig(BaseModel):
    heartbeat: Optional[HeartbeatConfig] = None

class AgentsRunningConfig(BaseModel):
    max_iters: int = 50         # ge=1
    max_input_length: int = 131072  # ge=1000

class AgentsConfig(BaseModel):
    defaults: AgentsDefaultsConfig
    running: AgentsRunningConfig
    language: str = "zh"
    installed_md_files_language: Optional[str] = None

class LastDispatchConfig(BaseModel):
    channel: str = ""
    user_id: str = ""
    session_id: str = ""

class MCPClientConfig(BaseModel):
    name: str                # 必填
    description: str = ""
    enabled: bool = True
    transport: Literal["stdio", "streamable_http", "sse"] = "stdio"
    url: str = ""
    headers: Dict[str, str] = {}
    command: str = ""
    args: List[str] = []
    env: Dict[str, str] = {}
    cwd: str = ""

class MCPConfig(BaseModel):
    clients: Dict[str, MCPClientConfig] = {}  # 字典，非 List

class Config(BaseModel):
    channels: ChannelConfig
    mcp: MCPConfig
    last_api: LastApiConfig
    agents: AgentsConfig
    last_dispatch: Optional[LastDispatchConfig] = None
    show_tool_details: bool = True
```

---

## 設定優先順序

當同一設定存在多個來源時，優先順序如下（由高到低）：

```
環境變數 > .env 檔案 > config.json > 程式碼預設值
```

---

## 常見設定場景

### 初始設定（開發環境）

```bash
# 1. 初始化工作目錄
copaw init --defaults

# 2. 設定 LLM Provider
# 透過前端 Settings → Models 設定，或直接編輯 ~/.copaw/providers.json

# 3. 啟動服務
copaw app
```

### 啟用 Telegram 頻道

1. 從 Telegram @BotFather 取得 Bot Token
2. 透過 API 或前端設定：
   ```http
   PUT /config/channels/telegram
   { "bot_token": "your-token", "show_typing": true }
   ```
3. 重啟服務使頻道生效

### 設定 Docker 環境

```bash
docker run -p 8088:8088 \
  -v copaw-data:/app/working \
  -e COPAW_WORKING_DIR=/app/working \
  -e DASHSCOPE_API_KEY=sk-your-key \
  agentscope/copaw:latest
```
