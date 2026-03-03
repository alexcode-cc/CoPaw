# 系統架構文件

## 1. 整體架構概覽

CoPaw 採用**三層架構**設計：前端（React/TypeScript）、後端（FastAPI/Python）、以及基於檔案系統的持久化層。

```
┌──────────────────────────────────────────────────────────────┐
│                    前端 (React/TypeScript)                     │
│           console/ → Ant Design + AgentScope UI              │
│                    Vite 6 Build → port:8088                  │
├──────────────────────────────────────────────────────────────┤
│               後端 API (FastAPI + uvicorn)                    │
│                    預設 port: 8088                            │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              API Routers (FastAPI)                   │    │
│  │  /models  /config  /cron  /skills  /mcp  /envs       │    │
│  │  /workspace  /local-models  /console  /agent         │    │
│  └──────────────────────────────────────────────────────┘    │
├──────────────────────────────────────────────────────────────┤
│                     Agent 層                                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  CoPawAgent（繼承 ReActAgent）                        │    │
│  │   ├── 內建工具：shell, file_io, browser, screenshot  │    │
│  │   ├── Skills：動態從 active_skills/ 載入              │    │
│  │   ├── 記憶：CoPawInMemoryMemory + MemoryManager      │    │
│  │   ├── System Prompt：AGENTS.md + SOUL.md + PROFILE.md│    │
│  │   ├── Hooks：BootstrapHook + MemoryCompactionHook    │    │
│  │   └── MCP Clients：stdio/SSE/HTTP                   │    │
│  └──────────────────────────────────────────────────────┘    │
├──────────────────────────────────────────────────────────────┤
│                     Channel 層                               │
│   DingTalk │ Feishu │ QQ │ Discord │ Telegram │ iMessage    │
│                  Console Channel（WebSocket/SSE）             │
├──────────────────────────────────────────────────────────────┤
│                     排程層                                    │
│          CronManager → APScheduler → CronExecutor           │
│             內建 Heartbeat 機制（預設每 6h 執行）               │
├──────────────────────────────────────────────────────────────┤
│               持久化層（~/.copaw 檔案系統）                     │
│  config.json  providers.json  jobs.json  chats.json         │
│  AGENTS.md  SOUL.md  PROFILE.md  MEMORY.md  HEARTBEAT.md   │
│  active_skills/  memory/  sessions/  models/                │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. 源碼目錄結構

### 2.1 專案根目錄

```
co-paw/
├── pyproject.toml          # Python 套件配置（PEP 517/518）
├── setup.py                # setuptools 入口
├── README.md               # 英文說明文件
├── README_zh.md            # 中文說明文件
├── CONTRIBUTING.md         # 英文貢獻指南
├── CONTRIBUTING_zh.md      # 中文貢獻指南
├── SECURITY.md             # 安全政策
├── .flake8                 # Flake8 程式碼品質設定
├── .pre-commit-config.yaml # Pre-commit hooks 設定
├── src/copaw/              # Python 主要源碼（核心套件）
├── console/                # 前端 React/TypeScript 應用
├── website/                # 文件網站（Vite/React）
├── scripts/                # 建置與安裝腳本
├── deploy/                 # 部署設定（Supervisor）
├── tests/                  # 測試檔案
└── .github/                # GitHub Actions CI/CD 工作流程
```

### 2.2 Python 源碼（`src/copaw/`）

```
src/copaw/
├── __init__.py
├── __version__.py              # 版本: "0.0.5b1"
├── constant.py                 # 全域常數（路徑、環境變數）
│
├── agents/                     # Agent 核心層
│   ├── react_agent.py          # CoPawAgent（繼承 ReActAgent）
│   ├── prompt.py               # System Prompt 建構
│   ├── command_handler.py      # /compact、/new 等系統指令
│   ├── schema.py               # Agent 資料結構
│   ├── skills_manager.py       # Skills 管理服務（SkillService）
│   ├── skills_hub.py           # Skills Hub 安裝
│   ├── hooks/
│   │   ├── bootstrap.py        # BootstrapHook（首次引導）
│   │   └── memory_compaction.py # MemoryCompactionHook（記憶壓縮）
│   ├── memory/
│   │   ├── copaw_memory.py     # CoPawInMemoryMemory（帶摘要支援）
│   │   ├── memory_manager.py   # MemoryManager（繼承 ReMeFb）
│   │   └── agent_md_manager.py # AGENTS.md 管理
│   ├── tools/
│   │   ├── __init__.py         # 工具匯出（含未註冊工具）
│   │   ├── file_io.py          # read_file / write_file / edit_file / append_file
│   │   ├── file_search.py      # grep_search / glob_search（有匯出但未註冊到 Agent）
│   │   ├── shell.py            # execute_shell_command
│   │   ├── send_file.py        # send_file_to_user
│   │   ├── browser_control.py  # browser_use（Playwright 多 action 控制）
│   │   ├── browser_snapshot.py # 瀏覽器快照
│   │   ├── desktop_screenshot.py # 桌面截圖（mss）
│   │   ├── get_current_time.py # 取得當前時間
│   │   └── memory_search.py    # create_memory_search_tool（動態建立）
│   ├── utils/
│   │   ├── file_handling.py
│   │   ├── message_processing.py
│   │   ├── token_counting.py
│   │   └── tool_message_utils.py
│   ├── md_files/
│   │   ├── en/                 # 英文 MD 範本
│   │   └── zh/                 # 中文 MD 範本
│   └── skills/                 # 內建 Skills
│       ├── cron/SKILL.md
│       ├── pdf/
│       ├── docx/
│       ├── xlsx/
│       ├── pptx/
│       ├── news/SKILL.md
│       ├── browser_visible/SKILL.md
│       ├── file_reader/SKILL.md
│       ├── himalaya/
│       └── dingtalk_channel/SKILL.md
│
├── app/                        # FastAPI 應用程式層
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── agent.py
│   │   ├── providers.py
│   │   ├── config.py
│   │   ├── skills.py
│   │   ├── mcp.py
│   │   ├── envs.py
│   │   ├── workspace.py
│   │   ├── local_models.py
│   │   ├── ollama_models.py
│   │   └── console.py
│   ├── channels/
│   │   ├── base.py             # BaseChannel 抽象基礎類別
│   │   ├── manager.py          # ChannelManager
│   │   ├── registry.py         # 頻道註冊表
│   │   ├── renderer.py         # MessageRenderer
│   │   ├── dingtalk/
│   │   ├── feishu/
│   │   ├── qq/
│   │   ├── discord_/
│   │   ├── imessage/
│   │   └── console/            # Console 頻道（Web UI）
│   ├── crons/
│   │   ├── api.py
│   │   ├── manager.py          # CronManager（APScheduler）
│   │   ├── executor.py
│   │   ├── heartbeat.py
│   │   └── models.py           # CronJobSpec, CronJobState
│   ├── runner/
│   │   ├── runner.py           # AgentRunner（繼承 Runner）
│   │   ├── session.py          # SafeJSONSession（Windows 相容檔名）
│   │   ├── manager.py          # ChatManager（聊天 CRUD）
│   │   ├── models.py           # ChatSpec / ChatHistory / ChatsFile
│   │   ├── api.py              # Chats CRUD API（/api/chats）
│   │   ├── utils.py            # build_env_context / agentscope_msg_to_message
│   │   ├── query_error_dump.py # 查詢錯誤 dump（除錯用）
│   │   └── repo/               # Session 與 Chat 持久化（JSON）
│   ├── console_push_store.py   # Console 推播訊息暫存（上限 500 則 / 60 秒）
│   └── _app.py                 # FastAPI 應用入口（Lifespan 管理）
│
├── config/
│   ├── config.py               # Pydantic 設定模型
│   ├── watcher.py              # 設定檔監控
│   └── utils.py                # 設定讀寫工具
│
├── providers/
│   ├── models.py               # Provider 資料模型
│   └── store.py                # Provider 設定讀寫
│
├── envs/
│   └── store.py                # 環境變數讀寫
│
├── local_models/
│   ├── manager.py              # LocalModelManager（HuggingFace/ModelScope 下載）
│   ├── schema.py               # LocalModelInfo, BackendType
│   ├── chat_model.py           # 本地模型 ChatModel 包裝
│   ├── tag_parser.py           # 模型特殊標籤解析
│   └── backends/
│       ├── llamacpp_backend.py # llama.cpp 後端
│       └── mlx_backend.py      # MLX 後端（Apple Silicon）
│
├── cli/
│   ├── main.py                 # CLI 入口（click group）
│   ├── app_cmd.py              # copaw app（啟動服務）
│   ├── init_cmd.py             # copaw init（初始化）
│   ├── channels_cmd.py         # copaw channels
│   ├── chats_cmd.py            # copaw chats
│   ├── cron_cmd.py             # copaw cron
│   ├── env_cmd.py              # copaw env
│   ├── skills_cmd.py           # copaw skills
│   ├── providers_cmd.py        # copaw models
│   ├── clean_cmd.py            # copaw clean
│   ├── uninstall_cmd.py        # copaw uninstall
│   ├── http.py                 # CLI HTTP 客戶端
│   └── utils.py                # CLI 工具函式
│
└── tokenizer/                  # 本地 tokenizer 資源（merges.txt）
```

---

## 3. 核心模組說明

### 3.1 Agent 核心（`CoPawAgent`）

`CoPawAgent` 繼承自 AgentScope 的 `ReActAgent`，是整個系統的智能核心。

**初始化參數：**
```python
def __init__(
    self,
    env_context: Optional[str] = None,
    enable_memory_manager: bool = True,
    mcp_clients: Optional[List[Any]] = None,
    memory_manager: MemoryManager | None = None,
    max_iters: int = 50,
    max_input_length: int = 128 * 1024,  # 131072 tokens
)
```

**初始化流程（10 步）：**
1. 設定 `_env_context`、`_max_input_length`、`_mcp_clients`
2. 計算 `_memory_compact_threshold = max_input_length × MEMORY_COMPACT_RATIO`
3. 建立 Toolkit：`_create_toolkit()`（載入內建工具）
4. 註冊 Skills：`_register_skills(toolkit)`（從 `active_skills/` 動態載入）
5. 建立 System Prompt：`_build_sys_prompt()`（讀取 AGENTS/SOUL/PROFILE.md）
6. 建立 Model 與 Formatter：`create_model_and_formatter()`
7. 呼叫 `super().__init__()` 初始化 ReActAgent（預設名稱 "Friday"）
8. 設定 MemoryManager：`_setup_memory_manager()`（注入 `memory_search` 工具）
9. 建立 CommandHandler（處理 `/compact`、`/new` 等指令）
10. 註冊 Hooks：`_register_hooks()`（BootstrapHook + MemoryCompactionHook）

**主要方法：**

| 方法 | 說明 |
|------|------|
| `_create_toolkit()` | 建立並註冊所有內建工具 |
| `_register_skills(toolkit)` | 從 working_dir 載入 Skills |
| `_build_sys_prompt()` | 建構 System Prompt |
| `_setup_memory_manager()` | 設定記憶管理 |
| `_register_hooks()` | 註冊 pre_reasoning hooks |
| `rebuild_sys_prompt()` | 重建 System Prompt（設定變更時呼叫） |
| `register_mcp_clients()` | 非同步註冊 MCP 客戶端 |
| `reply(msg, structured_model)` | 覆寫 ReActAgent，處理檔案/媒體與指令 |

**ReAct 迴圈（推理-行動）：**
- 最大迭代次數：50（可透過設定調整 `max_iters`）
- 最大輸入長度：128K tokens（可透過設定調整 `max_input_length`）
- 每次迭代：LLM 推理 → 選擇工具 → 執行工具 → 觀察結果 → 繼續推理
- `tool_choice` 正規化：`None` + 有工具 → `"auto"`

### 3.2 Channel 層

`BaseChannel` 抽象類別定義了統一的頻道介面：

- **消息接收**：`build_agent_request_from_native()` 將原生 Payload 轉為 `AgentRequest`
- **消息發送**：`send()` 發送文字、`send_content_parts()` 發送多種內容
- **消息合併（Debounce）**：防止短時間多個訊息觸發多次 Agent
- **事件發送**：`send_event()` 發送 Runner Event
- **工廠方法**：`from_env()` / `from_config()` 建立頻道實例
- **自訂頻道**：從 `~/.copaw/custom_channels/` 載入（ChannelRegistry）

**ChannelManager 機制：**
- 每個頻道 **4 個 Worker**，Queue 上限 **1000**
- 同一 Session 的 Payload 會合併後一次處理
- 支援 `replace_channel()` 進行頻道熱更新

**支援頻道：**

| 頻道 | 類別 | 輸入方式 | 輸出方式 |
|------|------|---------|---------|
| Console | `ConsoleChannel` | `/agent/process` API | stdout + push_store |
| DingTalk | `DingTalkChannel` | Stream WebSocket | sessionWebhook POST |
| Feishu | `FeishuChannel` | lark-oapi WebSocket | Open API（post/image/file） |
| Discord | `DiscordChannel` | discord.py on_message | `channel.send()` / DM |
| Telegram | `TelegramChannel` | python-telegram-bot polling | Bot API |
| QQ | `QQChannel` | WebSocket | HTTP API（c2c/群組/頻道） |
| iMessage | `iMessageChannel` | SQLite 輪詢 chat.db | imsg CLI（macOS 限定） |

### 3.3 Runner 層（Session 管理）

`AgentRunner` 繼承自 `agentscope_runtime.engine.runner.Runner`，負責處理每個會話的查詢請求：

- **query_handler()**：建立 CoPawAgent → 載入 Session → 串流輸出 → 儲存 Session
- **ChatManager**：管理 ChatSpec 的 CRUD，`get_or_create_chat()` 自動建立新聊天
- **SafeJSONSession**：繼承 JSONSession，對 session_id/user_id 做檔名 sanitize（Windows 相容）
- Session 狀態持久化到 `sessions/<session_id>.json`
- **query_error_dump**：查詢出錯時將 traceback 和 agent state 寫入暫存 JSON 便於除錯

### 3.4 排程層（CronManager）

基於 APScheduler（AsyncIOScheduler）實現的 Cron 排程系統：

```
CronJobSpec（定義）
    ↓
CronManager.add_job()
    ↓
APScheduler AsyncIOScheduler（排程引擎）
    ↓ （觸發時，每 job 有 concurrency semaphore）
CronExecutor.execute()
    ├── task_type="text" → ChannelManager.send_text()（直接發送文字）
    └── task_type="agent" → Runner.stream_query() + ChannelManager.send_event()
```

**CronJobSpec 核心欄位：**
- `task_type`：`"text"` 或 `"agent"`（兩種任務類型）
- `schedule`：Cron 表達式（5 欄位）+ timezone
- `dispatch`：channel + target（user_id / session_id）+ meta
- `runtime`：max_concurrency + misfire_grace_seconds

**Heartbeat 機制：**
- 預設每 6 小時觸發一次（`HEARTBEAT_DEFAULT_EVERY = "6h"`）
- 以 `HEARTBEAT.md` 的內容作為 Agent 的指令
- `parse_heartbeat_every()` 解析 `"30m"`、`"1h"` 等時間格式
- `_in_active_hours()` 檢查活躍時段限制（如只在 08:00-22:00 執行）
- `target="last"` 時自動 dispatch 到上次對話的 Channel
- 結果發送到指定的 Channel（預設 `"main"`）

---

## 4. 資料流

### 4.1 使用者訊息處理流程

```
使用者輸入（任一頻道）
    ↓
Channel.build_agent_request_from_native()
    ↓
ChannelManager.enqueue(channel_id, payload)（thread-safe，4 workers）
    ↓
ChannelManager.consume_one()（同 session payload 合併處理）
    ↓
AgentRunner.query_handler(session_id, message)
    ↓
CommandHandler.is_command()（檢查 /compact、/new、/clear、/history 等）
    ↓
CoPawAgent.reply()（ReAct 迴圈）
    ├── BootstrapHook（首次互動引導）
    ├── MemoryCompactionHook（token 超限時壓縮）
    ├── 工具呼叫（shell / browser / file_io / memory_search / skills ...）
    └── 生成最終回覆
    ↓
Channel.send() / send_content_parts()（回覆給使用者）
```

### 4.2 設定熱重載流程

```
使用者透過 API/前端修改設定
    ↓
PUT /api/config/xxx 或 PUT /api/models/xxx
    ↓
config.json / providers.json 更新
    ↓
ConfigWatcher 輪詢偵測到檔案變更（每 2 秒）
    ↓
自動重載 channels 與 heartbeat 設定
```

### 4.3 應用啟動與關閉流程

**啟動順序（Lifespan）：**
```
1. AgentRunner.start()
2. MCP Manager 啟動
3. ChannelManager 啟動（各頻道 4 workers）
4. CronManager 啟動（載入 jobs + 註冊 heartbeat）
5. ChatManager 啟動
6. ConfigWatcher 啟動（輪詢 config.json）
7. MCPConfigWatcher 啟動
```

**關閉順序：**
```
1. ConfigWatcher 停止
2. MCPConfigWatcher 停止
3. CronManager 停止
4. ChannelManager 停止
5. MCP 關閉
6. AgentRunner 停止
```

---

## 5. 關鍵設計決策

### 5.1 無傳統資料庫設計

CoPaw 刻意**不使用關聯式資料庫**，所有設定、記憶、Session 均以 JSON/Markdown 檔案形式存放。

**優點：**
- 部署極度簡單，無需 DB 安裝與設定
- 使用者可直接查看、編輯任何資料
- 備份與遷移只需複製目錄
- MEMORY.md、AGENTS.md 等檔案對 LLM 友好

**缺點：**
- 不適合高並發或多使用者場景
- 大量資料時查詢效能較差

### 5.2 Skills 即能力（Markdown 驅動）

Skills 採用 SKILL.md 驅動的設計，每個 Skill 是一個 Markdown 文件加上可選腳本：

- **無需修改核心程式碼**即可擴展 Agent 能力
- 非工程師也能撰寫 Skill（只需 Markdown）
- 透過三層架構（內建 → 自訂 → 啟用）支援覆蓋與客製化

### 5.3 ReAct + 工具呼叫

基於 AgentScope 的 `ReActAgent` 採用 ReAct（推理-行動）迴圈：
- **優點**：Agent 能自主決定使用哪些工具組合完成任務
- **限制**：最多 50 次迭代，超過後強制結束（防止無限迴圈）

### 5.4 MCP 協議整合

支援 Model Context Protocol（MCP），可整合大量外部工具：
- **stdio**：本地子進程（最常見）
- **SSE**：Server-Sent Events（遠端服務）
- **streamable_http**：HTTP streaming（新標準）

預設整合 `tavily_search` MCP 客戶端（需 `TAVILY_API_KEY`）。
