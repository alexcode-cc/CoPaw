# Skills 系統文件

Skills 是 CoPaw 的**能力擴展機制**，讓 Agent 能夠執行特定領域的複雜任務。每個 Skill 是一個包含 `SKILL.md` 的資料夾，無需修改核心程式碼即可擴展 Agent 能力。

---

## 目錄

- [Skills 概念](#skills-概念)
- [三層目錄架構](#三層目錄架構)
- [SKILL.md 格式規範](#skillmd-格式規範)
- [內建 Skills 清單](#內建-skills-清單)
- [自訂 Skill 開發指南](#自訂-skill-開發指南)
- [Skills Hub（遠端安裝）](#skills-hub遠端安裝)
- [Skills 管理 API](#skills-管理-api)
- [Skills 同步機制](#skills-同步機制)
- [Skills 載入原理](#skills-載入原理)

---

## Skills 概念

### Skill 是什麼？

一個 Skill 是一個**資料夾**，包含：
- `SKILL.md`（必要）：YAML Front Matter + 使用說明 + 工具定義
- `references/`（可選）：參考文件、API 文件、範例
- `scripts/`（可選）：Python 或 Shell 腳本

### Skills vs 內建工具的區別

| | 內建工具（Tools） | Skills |
|-|-----------------|--------|
| 定義方式 | Python 程式碼 | Markdown 文件 |
| 修改方式 | 需修改源碼 | 只需編輯 MD 文件 |
| 使用對象 | 開發者 | 任何人 |
| 載入機制 | 啟動時固定載入 | 可動態啟用/停用 |
| 典型用途 | 基礎操作（shell、file、browser） | 領域知識（PDF、Excel、郵件） |

### Skills 工作原理

Agent 在 System Prompt 中注入所有已啟用 Skill 的 `SKILL.md` 內容，讓 LLM 了解可用的能力。當用戶發送相關請求時，Agent 會按照 SKILL.md 中的使用說明執行對應操作。

### SkillInfo 資料模型

```python
class SkillInfo(BaseModel):
    name: str               # Skill 名稱
    content: str            # SKILL.md 完整內容
    source: str             # "builtin" | "customized" | "active"
    path: str               # 檔案系統路徑
    references: dict = {}   # references/ 目錄下的檔案
    scripts: dict = {}      # scripts/ 目錄下的檔案
```

---

## 三層目錄架構

```
優先順序（高 → 低）：
customized_skills/ > 內建 skills/ > （不存在）

載入順序：
active_skills/ ← 從 customized_skills/ 或 内建 skills/ 同步過來
```

### 三層說明

| 層次 | 路徑 | 說明 |
|------|------|------|
| 內建 Skills | `src/copaw/agents/skills/` | 隨套件內建，受版本管控，無法刪除 |
| 自訂 Skills | `~/.copaw/customized_skills/` | 使用者自建或從 Hub 安裝 |
| 啟用 Skills | `~/.copaw/active_skills/` | Agent 實際載入的 Skills（由前兩層同步） |

### 優先順序規則

當同一個 Skill 名稱在「自訂」和「內建」中都存在時，**自訂 Skill 優先**（會覆蓋內建版本）。這讓使用者可以修改內建 Skill 的行為而不需要 fork 整個專案。

---

## SKILL.md 格式規範

每個 Skill 的 `SKILL.md` 使用 YAML Front Matter + Markdown 內容：

```markdown
---
name: my_skill
description: 一句話描述此 Skill 的功能
---

# My Skill 使用說明

## 可用功能

### 功能一：做某件事
- 何時使用：xxx
- 如何使用：xxx
- 注意事項：xxx

## 工具說明

### tool_name_one
- **用途**：xxx
- **參數**：
  - `param1`（string）：xxx
  - `param2`（int，可選）：xxx，預設為 0
- **範例**：
  ```
  工具呼叫範例
  ```

## 最佳實踐

- 建議一
- 建議二
```

### YAML Front Matter 欄位

| 欄位 | 說明 | 必填 |
|------|------|------|
| `name` | Skill 唯一識別名稱（小寫字母、數字、底線） | 是 |
| `description` | 一句話描述，顯示在 Skills 列表中 | 是 |

> **驗證**：`create_skill()` 會檢查 SKILL.md 是否包含有效的 YAML frontmatter，並且**必須**包含 `name` 和 `description` 兩個欄位，否則建立失敗。

---

## 內建 Skills 清單

### 1. `cron` — 定時任務管理

讓 Agent 能透過自然語言創建和管理 Cron 排程任務。

**典型使用場景：**
- 「每天早上 8 點提醒我查看郵件」
- 「每週一整理上週工作摘要」

### 2. `pdf` — PDF 文件處理

提供完整的 PDF 讀寫能力，包含表單填寫與結構驗證。

**包含工具：**
- `read_pdf`：讀取 PDF 文字內容
- `write_pdf`：建立 PDF 文件
- `fill_pdf_form`：填寫 PDF 表單欄位
- `validate_pdf`：驗證 PDF 結構完整性

### 3. `docx` — Word 文件處理

Microsoft Word 格式文件的讀寫操作，支援修訂追蹤。

**包含工具：**
- `read_docx`：讀取 Word 文件
- `write_docx`：建立或修改 Word 文件
- `track_changes_docx`：顯示修訂追蹤內容

### 4. `xlsx` — Excel 試算表處理

Excel 格式的讀取、寫入與公式計算。

**包含工具：**
- `read_xlsx`：讀取 Excel 資料
- `write_xlsx`：寫入 Excel
- `recalculate_xlsx`：觸發公式重計算

### 5. `pptx` — PowerPoint 簡報處理

建立與編輯 PowerPoint 簡報檔案。

**包含工具：**
- `create_pptx`：建立新簡報
- `read_pptx`：讀取簡報內容
- `edit_pptx_slide`：編輯特定投影片

### 6. `news` — 新聞摘要

從多個平台抓取並整理新聞內容。

**支援來源：**
- 小紅書、知乎
- Reddit
- B 站（bilibili）
- YouTube

**典型使用場景：**
- 「幫我整理今天 AI 領域的新聞」
- 「搜尋 Reddit 上關於 Python 的熱門討論」

### 7. `browser_visible` — 可視化瀏覽器操作

在真實可見的瀏覽器視窗中執行操作（非 headless 模式），便於觀察與除錯。

**vs 內建 `browser_use` 工具的區別：**
- 內建工具：Headless 模式（無視覺化介面）
- `browser_visible` Skill：開啟真實瀏覽器視窗

### 8. `file_reader` — 多格式檔案讀取

自動偵測並讀取各種格式的檔案內容。

**支援格式：**
- 文字格式：txt、md、json、yaml、toml、ini、csv
- 辦公格式：docx、xlsx、pptx、pdf
- 程式碼格式：py、js、ts 等

### 9. `himalaya` — 郵件客戶端

透過 himalaya CLI 工具管理電子郵件。

**包含工具：**
- 列出收件箱
- 讀取郵件
- 撰寫並發送郵件
- 搜尋郵件

**前置需求：** 需安裝 [himalaya](https://github.com/soywod/himalaya) CLI 工具。

### 10. `dingtalk_channel` — 鈐釘釘頻道操作

在鈐釘釘頻道內進行進階操作（需已設定 DingTalk 頻道）。

---

## 自訂 Skill 開發指南

### 步驟一：建立 Skill 資料夾

在 `~/.copaw/customized_skills/` 下建立新資料夾：

```bash
mkdir ~/.copaw/customized_skills/my_awesome_skill
```

### 步驟二：撰寫 SKILL.md

建立 `~/.copaw/customized_skills/my_awesome_skill/SKILL.md`：

```markdown
---
name: my_awesome_skill
description: 執行某項自訂的特殊功能
---

# My Awesome Skill

## 概述

此 Skill 提供 [具體功能描述] 的能力。

## 使用方式

當使用者需要 [某類任務] 時，使用此 Skill。

### 基本流程

1. 首先確認 [前置條件]
2. 使用 `[tool_name]` 執行 [操作]
3. 若 [條件 A]，則執行 [步驟 X]
4. 最後輸出 [結果格式]

## 可用指令

本 Skill 透過 `execute_shell_command` 執行以下指令：

```bash
# 說明指令用途
my_tool --option value
```

## 注意事項

- 注意事項一
- 注意事項二

## 範例

**使用者**：幫我 [具體任務]
**Agent 行為**：使用 execute_shell_command 執行 `my_tool ...`，然後...
```

### 步驟三：新增輔助腳本（可選）

若 Skill 需要 Python 腳本輔助：

```
~/.copaw/customized_skills/my_awesome_skill/
├── SKILL.md
├── scripts/
│   ├── process_data.py
│   └── helpers.py
└── references/
    └── api_docs.md
```

在 SKILL.md 中引用腳本：

```markdown
## 使用方式

使用 `execute_shell_command` 執行：
```bash
python ~/.copaw/customized_skills/my_awesome_skill/scripts/process_data.py --input {file_path}
```
```

### 步驟四：啟用 Skill

透過 API 或前端啟用：

```http
POST /skills/my_awesome_skill/enable
```

或透過 CLI：

```bash
copaw skills enable my_awesome_skill
```

### 步驟五：驗證

啟用後，透過前端聊天介面測試 Skill 是否正常運作：

```
用戶：測試一下 my_awesome_skill
```

---

## Skills Hub（遠端安裝）

CoPaw 支援從遠端 GitHub 倉庫安裝 Skills。

### 安裝方式

#### 透過 API

```http
POST /skills/hub/install
Content-Type: application/json

{
  "url": "https://github.com/username/repo/tree/main/skill_folder"
}
```

#### 透過 CLI

```bash
copaw skills install https://github.com/username/repo/tree/main/skill_folder
```

### 支援的 URL 格式

- GitHub 目錄：`https://github.com/user/repo/tree/branch/path/to/skill`
- GitHub Raw：`https://raw.githubusercontent.com/user/repo/branch/path/SKILL.md`

### 安裝位置

從 Hub 安裝的 Skills 存放於 `~/.copaw/customized_skills/`，因此具有最高優先順序，可覆蓋同名的內建 Skill。

---

## Skills 管理 API

### SkillService 靜態方法

| 方法 | 說明 |
|------|------|
| `list_all_skills()` | 列出 builtin + customized 所有 Skills |
| `list_available_skills()` | 列出 active 中已啟用的 Skills |
| `create_skill(name, content, ...)` | 建立自訂 Skill（支援 overwrite、references、scripts、extra_files） |
| `enable_skill(name, force?)` | 同步到 active（啟用） |
| `disable_skill(name)` | 從 active 移除（停用） |
| `delete_skill(name)` | 從 customized 刪除 |
| `sync_from_active_to_customized()` | 從 active 同步回 customized |
| `load_skill_file(skill_name, file_path, source)` | 讀取 references/scripts 檔案 |

### 底層同步函式

| 函式 | 說明 |
|------|------|
| `sync_skills_to_working_dir(skill_names?, force?)` | 從 builtin → active 同步 |
| `sync_skills_from_active_to_customized(skill_names?)` | 從 active → customized 同步 |
| `ensure_skills_initialized()` | 確認並記錄初始化狀態 |

詳細 API 端點請參考 [API 參考文件](./api-reference.md#skills-管理apiskills)。

### 快速參考

```bash
# 列出所有 Skills
GET /api/skills

# 列出已啟用的 Skills
GET /api/skills/available

# 啟用單一 Skill
POST /api/skills/{skill_name}/enable

# 停用單一 Skill
POST /api/skills/{skill_name}/disable

# 批次操作
POST /api/skills/batch-enable
POST /api/skills/batch-disable

# 搜尋 Hub
GET /api/skills/hub/search?q=keyword

# 從 Hub 安裝
POST /api/skills/hub/install

# 建立自訂 Skill
POST /api/skills

# 讀取 Skill 檔案內容
GET /api/skills/{skill_name}/files/{source}/{file_path}

# 刪除自訂 Skill（不可刪除內建 Skill）
DELETE /api/skills/{skill_name}
```

---

## Skills 同步機制

### 啟用（Enable）流程

```
customized_skills/skill_name 存在？
    ├── 是 → 複製到 active_skills/skill_name（覆蓋內建版本）
    └── 否 → 從內建 skills/ 複製到 active_skills/skill_name
```

### 停用（Disable）流程

```
從 active_skills/ 移除 skill_name 資料夾
```

### 刪除（Delete）流程

```
只能刪除 customized_skills/skill_name
內建 skills/ 受到保護，無法透過 API 刪除
```

---

## Skills 載入原理

### Agent 初始化時

`CoPawAgent` 在建立時，透過 `SkillService` 掃描 `active_skills/` 目錄：

1. 讀取每個子目錄的 `SKILL.md`
2. 解析 YAML Front Matter 取得 `name` 和 `description`
3. 以 `register_agent_skill(skill_name, skill_md_content)` 注入到 Agent

### 注入方式

SKILL.md 的完整內容被附加到 System Prompt 的特定區段，讓 LLM 在每次對話時都能看到所有已啟用 Skill 的使用說明。

### 動態重載

目前 Skills 的啟用/停用不需要重啟服務，但要讓新的 Skills 在對話中生效，需要開始一個新的對話 Session（`/new` 指令或重新整理頁面）。

---

## 最佳實踐

### 撰寫高品質 SKILL.md 的建議

1. **描述要具體**：明確說明「何時使用」和「如何使用」
2. **提供範例**：用具體例子說明工具呼叫方式
3. **說明限制**：明確指出 Skill 的限制和注意事項
4. **保持簡潔**：過長的說明會消耗更多 Token，降低效能
5. **使用中文**：若 Agent 語言設定為 `zh`，中文描述效果更好

### Skill 命名規則

- 使用小寫英文字母、數字和底線
- 避免與內建 Skill 同名（除非你打算覆蓋它）
- 使用有意義的名稱，如 `email_manager`、`github_helper`
