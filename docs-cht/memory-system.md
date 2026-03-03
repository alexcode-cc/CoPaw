# 記憶體管理系統文件

CoPaw 採用**雙層記憶架構**：短期記憶（對話歷史）與長期記憶（向量搜尋），並透過記憶壓縮機制管理 Token 使用量。

---

## 目錄

- [架構概覽](#架構概覽)
- [短期記憶（CoPawInMemoryMemory）](#短期記憶copawinnmemorymemory)
- [長期記憶（MemoryManager）](#長期記憶memorymanager)
- [記憶壓縮機制](#記憶壓縮機制)
- [Markdown 記憶檔案](#markdown-記憶檔案)
- [記憶搜尋工具](#記憶搜尋工具)
- [Session 持久化](#session-持久化)
- [環境變數設定](#環境變數設定)
- [常見問題](#常見問題)

---

## 架構概覽

```
┌─────────────────────────────────────────────────────┐
│                   記憶體管理系統                       │
├─────────────────────────────────────────────────────┤
│  短期記憶（CoPawInMemoryMemory）                       │
│  ┌───────────────────────────────────────────────┐  │
│  │  對話歷史（messages list）                     │  │
│  │  ├── Msg 1: 使用者輸入                         │  │
│  │  ├── Msg 2: 工具呼叫                           │  │
│  │  ├── Msg 3: 工具回傳                           │  │
│  │  └── Msg N: Agent 回覆                        │  │
│  │                                               │  │
│  │  壓縮摘要（_compressed_summary）              │  │
│  │  （當 token 超限時，舊對話被摘要壓縮）          │  │
│  └───────────────────────────────────────────────┘  │
│                        ↑↓                           │
│  MemoryCompactionHook（pre_reasoning 自動觸發壓縮）     │
├─────────────────────────────────────────────────────┤
│  長期記憶（MemoryManager / ReMeFb）                   │
│  ┌───────────────────────────────────────────────┐  │
│  │  監控檔案：                                    │  │
│  │  ├── MEMORY.md（精選長期記憶）                  │  │
│  │  └── memory/YYYY-MM-DD.md（每日日記）           │  │
│  │                                               │  │
│  │  索引引擎：                                    │  │
│  │  ├── 向量搜尋（FAISS/Chroma + Embedding API）  │  │
│  │  └── 全文搜尋（FTS）                           │  │
│  └───────────────────────────────────────────────┘  │
│                        ↑                            │
│  memory_search 工具（Agent 主動查詢）                  │
├─────────────────────────────────────────────────────┤
│  Session 持久化                                       │
│  sessions/<session_id>.json（序列化的對話狀態）         │
└─────────────────────────────────────────────────────┘
```

---

## 短期記憶（`CoPawInMemoryMemory`）

### 定義位置

`src/copaw/agents/memory/copaw_memory.py`

### 繼承關係

```
InMemoryMemory（agentscope 內建）
    └── CoPawInMemoryMemory（CoPaw 擴展）
```

### 核心資料結構

```python
class CoPawInMemoryMemory(InMemoryMemory):
    # 對話歷史：(Msg, marks) 的列表
    # marks 使用 _MemoryMark 枚舉標記狀態（如 COMPRESSED）
    _memory: List[Tuple[Msg, Dict]]

    # 壓縮摘要（壓縮後的歷史摘要文字）
    _compressed_summary: Optional[str]
```

### 主要方法

| 方法 | 說明 |
|------|------|
| `get_memory(mark?, exclude_mark?, prepend_summary?)` | 取得記憶（支援按標記過濾、摘要前置） |
| `get_compressed_summary()` | 取得壓縮後的摘要文字 |
| `state_dict()` | 序列化為 dict（用於 Session 持久化） |
| `load_state_dict()` | 從 dict 反序列化（恢復 Session） |

### 工作原理

1. **每輪對話**：所有訊息（使用者輸入、工具呼叫/結果、Agent 回覆）都存入 `_memory` 列表
2. **Token 計算**：使用本地 BPE tokenizer（`tokenizer/merges.txt`）計算累積 Token 數
3. **標記管理**：使用 `_MemoryMark.COMPRESSED` 標記已壓縮的訊息
4. **壓縮觸發**：當 Token 數超過 `max_input_length × compact_ratio`（預設 0.7）時觸發
5. **壓縮後**：舊對話被 LLM 摘要為 `_compressed_summary`，保留最近 N 輪原始對話
6. **摘要前置**：`prepend_summary=True` 時，壓縮摘要自動加在記憶前面

### 訊息格式（`Msg`）

```python
class Msg:
    name: str           # 發言者名稱
    role: str           # "user" | "assistant" | "system" | "tool"
    content: Any        # 訊息內容（文字或結構化資料）
    tool_calls: List    # 工具呼叫請求（assistant 角色）
    tool_call_id: str   # 工具呼叫 ID（tool 角色）
```

---

## 長期記憶（`MemoryManager`）

### 定義位置

`src/copaw/agents/memory/memory_manager.py`

### 繼承關係

```
ReMeFb（reme-ai 套件）
    └── MemoryManager（CoPaw 擴展）
```

### 主要方法

| 方法 | 說明 |
|------|------|
| `compact_memory()` | 壓縮訊息記憶 |
| `summary_memory()` | 產生記憶摘要 |
| `memory_search()` | 語意搜尋長期記憶 |
| `memory_get()` | 讀取記憶檔片段 |
| `add_async_summary_task()` | 建立背景摘要任務 |
| `await_summary_tasks()` | 等待所有摘要任務完成 |

### 工作原理

`MemoryManager` 透過**監控 Markdown 檔案**建立記憶索引：

1. 監控 `MEMORY.md` 和 `memory/YYYY-MM-DD.md` 的檔案變更
2. 自動解析 Markdown 內容並分割為記憶片段
3. 使用 Embedding API 將記憶片段向量化
4. 儲存到向量資料庫（Chroma 或本地 FAISS）
5. 同時建立全文搜尋索引（FTS）

### AgentMdManager

`AgentMdManager` 負責管理工作目錄與 memory 目錄的 Markdown 檔案 I/O：

| 方法 | 說明 |
|------|------|
| `list_working_mds()` | 列出工作目錄的 MD 檔案 |
| `read_working_md(name)` | 讀取工作目錄的 MD 檔案 |
| `write_working_md(name, content)` | 寫入工作目錄的 MD 檔案 |
| `list_memory_mds()` | 列出 memory/ 目錄的 MD 檔案 |
| `read_memory_md(name)` | 讀取 memory/ 目錄的 MD 檔案 |
| `write_memory_md(name, content)` | 寫入 memory/ 目錄的 MD 檔案 |

> Agent Router（`/api/agent/files/*`、`/api/agent/memory/*`）即是透過 AgentMdManager 操作這些檔案。

### 向量後端

| 後端 | 適用場景 | 說明 |
|------|---------|------|
| `chroma` | Linux/macOS（預設） | 需要 ChromaDB，搜尋效能較好 |
| `local` | Windows（預設）、輕量部署 | 使用本地 FAISS，無需額外服務 |
| `auto` | 自動選擇 | Windows 選 local，其他選 chroma |

設定方式：
```bash
export MEMORY_STORE_BACKEND=local  # 強制使用本地後端
```

### Embedding 設定

預設使用阿里雲 DashScope 的 `text-embedding-v4` 模型：

```bash
export EMBEDDING_API_KEY=sk-your-key
export EMBEDDING_BASE_URL=https://dashscope.aliyuncs.com/...
export EMBEDDING_MODEL_NAME=text-embedding-v4
export EMBEDDING_DIMENSIONS=1024
```

**停用長期記憶：**
```bash
export ENABLE_MEMORY_MANAGER=false
```

---

## 記憶壓縮機制

### 觸發條件

當當前對話的 Token 數超過以下閾值時，`MemoryCompactionHook` 自動觸發壓縮：

```
觸發閾值 = max_input_length × COPAW_MEMORY_COMPACT_RATIO
```

預設值：`131072 × 0.7 ≈ 91,750 tokens`

### MemoryCompactionHook

**檔案位置**：`src/copaw/agents/hooks/memory_compaction.py`（獨立檔案，非 `bootstrap.py`）

**觸發時機**：`pre_reasoning` 階段，每次 Agent 推理前自動檢查。

**額外功能**：支援 `ENABLE_TRUNCATE_TOOL_RESULT_TEXTS` 環境變數，可截斷過長的 tool_result 文字以節省 Token。

### 壓縮流程

```
1. 計算目前所有對話的 Token 總數
2. 判斷是否超過觸發閾值
   │
   ├── 未超過 → 不執行壓縮，繼續正常對話
   │
   └── 超過 → 開始壓縮
       │
       ├── 保留 system prompt + 最近 keep_recent 則訊息
       │   （keep_recent = COPAW_MEMORY_COMPACT_KEEP_RECENT，預設 3）
       │
       ├── 對其餘舊對話呼叫 LLM 生成摘要
       │   （摘要 Prompt 要求保留關鍵資訊和決策）
       │
       ├── 將摘要存入 _compressed_summary
       │
       ├── 使用 _MemoryMark.COMPRESSED 標記已壓縮訊息
       │
       └── 從 _memory 移除已壓縮的舊對話
           （保留 _compressed_summary + 最近 N 輪原始對話）
```

### 壓縮後的 Context 結構

```
[系統 Prompt]
[壓縮摘要]
[最近 N 輪原始對話]
[當前輸入]
```

### 調整壓縮參數

```bash
# 觸發壓縮的 Token 占比（預設 0.7，即 70%）
export COPAW_MEMORY_COMPACT_RATIO=0.8

# 壓縮後保留最近 N 輪原始對話（預設 3）
export COPAW_MEMORY_COMPACT_KEEP_RECENT=5

# 可選：截斷過長的 tool_result 文字
export ENABLE_TRUNCATE_TOOL_RESULT_TEXTS=true
```

### 手動觸發壓縮與相關指令

CoPaw 的 `CommandHandler` 支援以下 `/` 開頭的系統指令：

| 指令 | 說明 |
|------|------|
| `/compact` | 手動觸發記憶壓縮 |
| `/new` 或 `/start` | 清空摘要並開啟新對話，背景產生摘要 |
| `/clear` | 完全清空記憶與壓縮摘要 |
| `/history` | 顯示訊息歷史、Token 估計、Context 使用率 |
| `/compact_str` | 顯示目前的壓縮摘要內容 |
| `/await_summary` | 等待所有背景摘要任務完成 |

---

## Markdown 記憶檔案

CoPaw 使用 Markdown 檔案作為長期記憶的「倉庫」，這是 CoPaw 的核心設計哲學之一：**記憶對人類可讀可編輯**。

### MEMORY.md — 精選長期記憶

`~/.copaw/MEMORY.md` 儲存 Agent 的核心長期記憶，可由：
- 使用者手動添加重要資訊
- Agent 主動更新（當 Agent 學到重要新知識時）

**範例內容：**

```markdown
# 記憶

## 個人偏好
- 用戶喜歡用繁體中文溝通
- 用戶偏好簡潔的回答，不需要過多解釋
- 用戶的工作時間是週一到週五 09:00-18:00

## 重要資訊
- 主要工作專案：CoPaw（在 ~/ws/co-paw）
- 常用的 Python 環境：~/envs/copaw
- 資料庫連線：postgresql://localhost:5432/mydb

## 習慣與規則
- 程式碼提交前總是先執行測試
- 文件使用繁體中文撰寫
```

### memory/YYYY-MM-DD.md — 每日日記

每天的對話摘要和重要事件自動記錄到對應日期的 Markdown 檔案。

**範例：**`~/.copaw/memory/2026-03-03.md`

```markdown
# 2026-03-03

## 對話摘要

- 09:30 幫用戶分析 CoPaw 專案架構
- 11:00 修復前端聊天頁面的 bug
- 15:00 建立新的 Cron 任務（每日新聞摘要）

## 重要決策

- 決定使用 DashScope 作為主要 LLM Provider
- 將 MEMORY_STORE_BACKEND 設為 local

## 待辦事項

- [ ] 研究如何整合 Telegram 頻道
- [ ] 撰寫 Skills 開發指南
```

### HEARTBEAT.md — 心跳任務

`~/.copaw/HEARTBEAT.md` 定義了每次心跳觸發時 Agent 要執行的任務清單。

**範例內容：**

```markdown
# 心跳任務

每次心跳觸發時，請依序執行以下任務：

1. **檢查郵件**：使用 himalaya 工具查看是否有新的重要郵件
2. **日程提醒**：確認今天是否有重要的會議或截止日期
3. **快訊整理**：整理過去 6 小時的科技新聞摘要，發送到 Telegram
4. **系統狀態**：檢查服務器磁碟空間和 CPU 使用率

如有重要事項，請透過 Console 頻道通知用戶。
```

---

## 記憶搜尋工具

### `memory_search` 工具

當 `MemoryManager` 初始化成功後，Agent 會自動獲得一個 `memory_search` 工具。

**工具用途：** 搜尋長期記憶中與當前問題相關的過去資訊。

**搜尋方式：**
1. **語義搜尋**（向量相似度）：找到語意相近的記憶片段
2. **全文搜尋**（關鍵字）：精確搜尋特定詞彙

**Agent 使用時機：**
- 用戶提到過去曾討論的事情
- 需要查詢用戶的偏好設定
- 尋找歷史決策或重要資訊

**範例：**

```
用戶：上次我們說好的資料庫設定是什麼？

Agent 行為：
1. 呼叫 memory_search("資料庫設定")
2. 搜尋結果返回 MEMORY.md 中的相關片段
3. 使用這些資訊回答用戶
```

---

## Session 持久化

### Session 說明

每個對話 Session 對應一個獨立的對話歷史。Session ID 通常由頻道和使用者決定。

### 持久化機制

`SafeJSONSession` 負責 Session 狀態的序列化：

```
會話結束 / 中斷
    ↓
序列化 CoPawInMemoryMemory 到 JSON
    ↓
存儲到 ~/.copaw/sessions/<session_id>.json
    ↓
（下次開啟相同 Session 時）
    ↓
從 sessions/<session_id>.json 反序列化
    ↓
還原對話歷史（包括 _compressed_summary）
```

### Session JSON 格式

```json
{
  "session_id": "console_user_001",
  "created_at": "2026-03-03T09:00:00+08:00",
  "updated_at": "2026-03-03T15:30:00+08:00",
  "memory": {
    "messages": [...],
    "compressed_summary": "摘要文字...",
    "token_count": 45230
  }
}
```

### 清除 Session

```bash
# 透過 CLI 清除特定 Session
copaw chats delete <session_id>

# 透過 API
DELETE /agent/sessions/{session_id}

# 在聊天中使用指令開新對話（不刪除舊 Session）
/new
```

---

## 環境變數設定

| 環境變數 | 說明 | 預設值 |
|----------|------|--------|
| `ENABLE_MEMORY_MANAGER` | 是否啟用長期記憶管理 | `true` |
| `MEMORY_STORE_BACKEND` | 向量後端（`auto`/`local`/`chroma`） | `auto` |
| `COPAW_MEMORY_COMPACT_RATIO` | 壓縮觸發比例 | `0.7` |
| `COPAW_MEMORY_COMPACT_KEEP_RECENT` | 壓縮後保留最近 N 輪 | `3` |
| `EMBEDDING_API_KEY` | 向量 Embedding API Key | - |
| `EMBEDDING_BASE_URL` | Embedding Base URL | DashScope URL |
| `EMBEDDING_MODEL_NAME` | Embedding 模型名稱 | `text-embedding-v4` |
| `EMBEDDING_DIMENSIONS` | Embedding 向量維度 | `1024` |

---

## 常見問題

### Q: 如何讓 Agent 記住某件重要的事？

直接在聊天中告訴 Agent，或手動編輯 `~/.copaw/MEMORY.md`：

```
用戶：請記住，我的 GitHub 帳號是 myusername

Agent 會：
1. 將此資訊寫入 MEMORY.md
2. 下次搜尋記憶時可以找到
```

### Q: Windows 上記憶搜尋不工作？

Windows 預設使用 `local`（FAISS）後端，需確認：
1. `EMBEDDING_API_KEY` 是否已設定
2. 若不想使用 Embedding，設定 `ENABLE_MEMORY_MANAGER=false`

### Q: 如何清除所有記憶？

```bash
# 清除長期記憶
rm ~/.copaw/MEMORY.md
rm -rf ~/.copaw/memory/

# 清除所有 Session 歷史
rm -rf ~/.copaw/sessions/
```

### Q: 記憶壓縮後，之前的對話還在嗎？

壓縮後的舊對話以**摘要形式**保留在 `_compressed_summary` 中，原始訊息被刪除。壓縮摘要會在 Session 序列化時一同儲存，重新開啟 Session 後仍可取得。

### Q: 如何避免頻繁的記憶壓縮？

提高觸發比例或增加保留輪數：
```bash
export COPAW_MEMORY_COMPACT_RATIO=0.9     # 90% 才觸發（預設 70%）
export COPAW_MEMORY_COMPACT_KEEP_RECENT=10 # 保留最近 10 輪（預設 3）
```
