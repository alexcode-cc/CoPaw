# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

總是以繁體中文回應。

## Project Overview

CoPaw (Co Personal Agent Workstation) 是一個可部署的個人 AI 助理框架，基於 AgentScope 的 ReActAgent，支援多頻道（Discord、DingTalk、Feishu、QQ、Telegram、iMessage）、可擴充 Skills、本地/雲端 LLM，所有資料儲存在本地 `~/.copaw/`。

## Development Commands

### Backend (Python)

```bash
pip install -e ".[dev]"              # 安裝開發依賴
pip install -e ".[dev,ollama,mlx]"   # 含本地模型支援
copaw app                            # 啟動 FastAPI 服務 (port 8088)
copaw init                           # 互動式初始化工作目錄
pytest                               # 執行測試
pytest -m "not slow"                 # 跳過慢速測試
pre-commit run --all-files           # 程式碼品質檢查 (black, flake8, pylint, mypy)
```

### Frontend (React/TypeScript)

```bash
cd console
npm ci                               # 安裝依賴
npm run dev                          # Vite 開發伺服器
npm run build                        # 生產版本建置 (tsc -b && vite build)
npm run lint                         # ESLint
npm run format                       # Prettier 格式化
```

### Build & Docker

```bash
bash scripts/wheel_build.sh          # 建置 Python wheel（含前端）
bash scripts/docker_build.sh [TAG]   # 建置 Docker image
```

## Architecture

```
src/copaw/
├── app/                 # FastAPI 應用層
│   ├── _app.py          # 主應用 + lifespan（啟動順序：config → MCP → channels → cron → watchers）
│   ├── channels/        # 頻道管理（registry.py 自動發現頻道實作）
│   ├── routers/         # REST API endpoints
│   ├── runner/          # Chat session 管理與訊息處理
│   ├── crons/           # APScheduler 排程任務
│   └── mcp/             # Model Context Protocol 客戶端
├── agents/              # Agent 核心
│   ├── react_agent.py   # CoPawAgent — 主 Agent（extends agentscope ReActAgent）
│   ├── skills_manager.py # Skill 三層同步（builtin → customized → active_skills）
│   ├── memory/          # 雙層記憶（短期 in-memory + 長期 ReMeFs 向量搜尋）
│   ├── tools/           # 內建工具（shell, file I/O, browser_use, screenshot）
│   └── skills/          # 內建 Skills（cron, pdf, docx, xlsx, pptx, news, himalaya...）
├── cli/                 # Click CLI 入口（main.py → 各 *_cmd.py 子命令）
├── config/              # Pydantic config 模型（支援 hot-reload watcher）
├── providers/           # LLM Provider 系統
│   ├── registry.py      # 內建 Provider 註冊（dashscope, openai, ollama, modelscope）
│   ├── store.py         # providers.json 持久化與 CRUD
│   └── ollama_manager.py # Ollama 模型管理
├── local_models/        # 本地模型（llama.cpp GGUF / MLX safetensors）
├── envs/                # 環境變數管理（.env 檔案操作）
└── constant.py          # 全域常數（工作目錄路徑、預設值）

console/                 # React 18 + TypeScript 5.8 + Vite 6 + Ant Design 前端
```

### Key Design Patterns

- **Agent 系統**：CoPawAgent 初始化 10 步（env → memory → toolkit → skills → prompt → model → parent → hooks），系統提示由 AGENTS.md + SOUL.md + PROFILE.md 組合。短期記憶在 token 達 `max_input_length × 0.7` 時自動壓縮。
- **Provider 系統**：分內建 (`providers`) 與自訂 (`custom_providers`)，統一使用 OpenAI 相容介面 (`OpenAIChatModel`)。Ollama 特殊處理：無需 API key 即視為已設定。
- **Channel 系統**：所有頻道繼承 `BaseChannel`，統一訊息協議 `AgentRequest` + `content_parts`。ChannelManager 每頻道 4 worker thread、1000 訊息佇列、payload debouncing。
- **Skills 系統**：每個 Skill 是一個含 `SKILL.md` 的目錄（YAML front matter 必須有 `name` 和 `description`），可選 `references/` 和 `scripts/` 子目錄。三層優先級：customized > builtin。
- **Config Hot-Reload**：channels 與 MCP clients 設定變更時自動重新載入，無需重啟服務。

### Data Flow

```
User → Channel → ChannelManager → AgentRunner → CoPawAgent (ReAct loop with tools/skills)
                                                     ↕ memory_search / memory_compact
Response ← Channel ← ChannelManager ← AgentRunner ← CoPawAgent
```

## Code Style & Conventions

- **Commit 格式**：Conventional Commits — `<type>(<scope>): <subject>`，type: feat/fix/docs/style/refactor/perf/test/chore
- **Python**：Black (79 字元行寬)，pre-commit 整合 flake8/pylint/mypy
- **Frontend**：ESLint + Prettier
- **非同步**：全面 async/await（FastAPI, channels, memory manager）
- **Python 版本**：3.10 ~ 3.13

## Working Directory Structure (~/.copaw/)

| 檔案 | 用途 |
|------|------|
| `config.json` | 頻道、MCP、Agent 行為設定 |
| `providers.json` | LLM 憑證與模型設定 |
| `AGENTS.md` / `SOUL.md` / `PROFILE.md` | Agent 系統提示組成 |
| `MEMORY.md` + `memory/` | 長期記憶（向量搜尋） |
| `HEARTBEAT.md` | 心跳排程任務指令 |
| `active_skills/` | Agent 載入的 Skills |
| `customized_skills/` | 使用者自訂 Skills（覆蓋 builtin） |

## Key Environment Variables

| 變數 | 用途 |
|------|------|
| `COPAW_WORKING_DIR` | 工作目錄（預設 `~/.copaw`） |
| `COPAW_LOG_LEVEL` | 日誌等級 |
| `COPAW_CORS_ORIGINS` | CORS 設定（開發用） |
| `COPAW_MEMORY_COMPACT_RATIO` | 記憶壓縮觸發比例（預設 0.7） |
| `COPAW_MEMORY_COMPACT_KEEP_RECENT` | 壓縮後保留最近訊息數（預設 3） |
| `ENABLE_MEMORY_MANAGER` | 啟用長期記憶向量搜尋 |
| `EMBEDDING_*` | Embedding API 設定 |
| `DASHSCOPE_API_KEY` | DashScope（通義千問）API key |
| `TAVILY_API_KEY` | 啟用預設 MCP 搜尋工具 |
