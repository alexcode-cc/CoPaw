# CoPaw 技術文件索引

> 本文件集為 CoPaw 專案的繁體中文技術分析文件，供開發者深入了解系統架構並繼續開發。

## 什麼是 CoPaw？

**CoPaw**（Co Personal Agent Workstation）是一個**個人 AI 助理框架**，由 AgentScope 團隊開發，採用 Apache 2.0 授權開源。

- **當前版本**：`0.0.5b1`
- **Python 支援**：3.10 ~ <3.14
- **PyPI 套件名稱**：`copaw`
- **官方網站**：https://copaw.agentscope.io
- **GitHub**：https://github.com/agentscope-ai/CoPaw

### 核心設計理念

> "Works for you, grows with you."（懂你所需，伴你左右）

CoPaw 是一個可部署在用戶自有機器或雲端的個人助理，透過「Skills」擴展能力、支援多頻道通訊，所有資料本地保存。

---

## 文件目錄

| 文件 | 說明 |
|------|------|
| [系統架構](./architecture.md) | 整體架構設計、模組關係、資料流、啟動/關閉流程 |
| [API 參考](./api-reference.md) | 後端 FastAPI 所有 `/api/*` 端點的完整說明 |
| [設定系統](./configuration.md) | config.json、頻道設定、MCP、Provider、Pydantic 模型完整定義 |
| [Skills 系統](./skills-system.md) | Skills 三層架構、SkillService、內建 Skills、自訂開發指南 |
| [記憶體管理](./memory-system.md) | 雙層記憶架構、壓縮機制、AgentMdManager、系統指令 |
| [前端開發指南](./frontend-guide.md) | React/TypeScript 技術棧、頁面路由、UI 元件、新增頁面 |
| [開發者指南](./developer-guide.md) | 開發環境、CLI 完整命令、新增 API/Channel/Tool、CI/CD、部署 |

---

## 快速導覽

### 想了解系統如何運作？
→ 閱讀 [系統架構](./architecture.md)（含啟動流程、資料流、架構圖）

### 想串接後端 API？
→ 閱讀 [API 參考](./api-reference.md)（所有 `/api/*` 端點，含回傳型別與範例）

### 想新增或修改設定功能？
→ 閱讀 [設定系統](./configuration.md)（含完整 Pydantic Model 定義與環境變數參考）

### 想開發新的 Skill（能力擴展）？
→ 閱讀 [Skills 系統](./skills-system.md)（含 SKILL.md 格式、SkillService API、Hub 安裝）

### 想了解 AI 記憶機制？
→ 閱讀 [記憶體管理](./memory-system.md)（含壓縮機制、系統指令、AgentMdManager）

### 想修改前端介面？
→ 閱讀 [前端開發指南](./frontend-guide.md)（含頁面路由、側欄結構、新增頁面步驟）

### 想建立開發環境或貢獻程式碼？
→ 閱讀 [開發者指南](./developer-guide.md)（含完整 CLI 命令、新增 Channel/Tool、CI/CD）

---

## 技術棧一覽

### 後端
| 技術 | 用途 |
|------|------|
| Python 3.10+ | 主要語言 |
| FastAPI + uvicorn | HTTP API 伺服器 |
| agentscope | ReActAgent 核心框架 |
| agentscope-runtime | Agent 執行期引擎 |
| APScheduler | 定時任務排程 |
| Playwright | 瀏覽器自動化 |
| reme-ai | 長期記憶向量搜尋 |
| Pydantic | 資料模型驗證 |

### 前端
| 技術 | 用途 |
|------|------|
| React 18 + TypeScript 5.8 | 主框架 |
| Ant Design 5 | UI 元件庫 |
| AgentScope Design System | 聊天介面元件 |
| React Router v7 | 前端路由 |
| Vite 6 | 建置工具 |
| react-i18next | 國際化 |

### 儲存方式
- **無傳統資料庫**：所有資料以 JSON/Markdown 檔案儲存於 `~/.copaw/`
- **記憶層**：向量搜尋（DashScope embedding）+ 全文搜尋（reme-ai）

### CLI 工具
- `copaw app`：啟動服務
- `copaw init`：初始化工作目錄
- `copaw channels`：管理頻道（安裝自訂頻道、設定）
- `copaw chats`：管理聊天 Session
- `copaw cron`：管理定時任務
- `copaw env`：管理環境變數
- `copaw models`：管理 LLM Provider 與本地模型
- `copaw skills`：管理 Skills
- `copaw clean`：清空工作目錄
- `copaw uninstall`：解除安裝

> 完整 CLI 命令參考請見 [開發者指南](./developer-guide.md#cli-命令完整參考)。

---

## 工作目錄結構（`~/.copaw/`）

```
~/.copaw/
├── config.json          # 主要設定（頻道、MCP、Agent 行為）
├── providers.json       # LLM Provider API Keys 與模型清單
├── jobs.json            # Cron Job 排程清單
├── chats.json           # 聊天對話清單
├── AGENTS.md            # Agent 工作流程與規則（System Prompt）
├── SOUL.md              # Agent 核心身份與行為原則
├── PROFILE.md           # Agent 與使用者個人資料
├── MEMORY.md            # 長期記憶（人工/AI 更新）
├── HEARTBEAT.md         # 心跳任務清單
├── active_skills/       # 當前啟用的 Skills
├── customized_skills/   # 使用者自訂 Skills
├── memory/              # 每日日記（YYYY-MM-DD.md）
├── sessions/            # 會話狀態（JSON）
├── models/              # 本地下載的 LLM 模型
└── custom_channels/     # 自訂頻道
```
