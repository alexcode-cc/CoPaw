# 開發者指南

本指南說明如何在本地建立開發環境、執行測試、以及理解 CI/CD 流程。

---

## 目錄

- [開發環境建立](#開發環境建立)
- [Python 後端開發](#python-後端開發)
- [CLI 命令完整參考](#cli-命令完整參考)
- [新增 API 端點](#新增-api-端點)
- [新增頻道（Channel）](#新增頻道channel)
- [新增內建工具（Tool）](#新增內建工具tool)
- [本地模型支援](#本地模型支援)
- [測試](#測試)
- [程式碼品質](#程式碼品質)
- [CI/CD 流程](#cicd-流程)
- [部署方式](#部署方式)
- [依賴項目說明](#依賴項目說明)
- [版本發布流程](#版本發布流程)

---

## 開發環境建立

### 前置需求

| 工具 | 版本 | 說明 |
|------|------|------|
| Python | 3.10 ~ 3.13 | 後端主要語言 |
| Node.js | 20+ | 前端建置工具 |
| npm | 10+ | 前端套件管理 |
| Git | 任意版本 | 版本控制 |

### 步驟一：Clone 專案

```bash
git clone https://github.com/agentscope-ai/CoPaw.git
cd co-paw
```

### 步驟二：建立 Python 虛擬環境

```bash
python -m venv .venv

# macOS/Linux
source .venv/bin/activate

# Windows (PowerShell)
.venv\Scripts\Activate.ps1
```

### 步驟三：安裝 Python 依賴（開發模式）

```bash
# 安裝核心依賴 + 開發工具
pip install -e ".[dev]"

# 若需要本地模型支援（擇一）：
pip install -e ".[dev,local]"      # HuggingFace 下載
pip install -e ".[dev,llamacpp]"   # llama.cpp 後端
pip install -e ".[dev,mlx]"        # Apple Silicon MLX
pip install -e ".[dev,ollama]"     # Ollama 整合
```

### 步驟四：安裝 Pre-commit Hooks

```bash
pre-commit install
```

### 步驟五：安裝前端依賴

```bash
cd console
npm install
cd ..
```

### 步驟六：初始化 CoPaw 工作目錄

```bash
copaw init --defaults
```

這會在 `~/.copaw/` 建立預設的工作目錄結構。

### 步驟七：設定 LLM Provider

編輯 `~/.copaw/providers.json` 或透過啟動後的前端設定：

```bash
# 啟動服務
copaw app

# 在瀏覽器開啟 http://127.0.0.1:8088
# 前往 Settings → Models 設定 API Key
```

---

## Python 後端開發

### 啟動後端（開發模式）

```bash
copaw app
# 或使用 --reload 參數啟用熱重載
copaw app --reload
# 或直接使用 uvicorn
uvicorn copaw.app._app:app --host 127.0.0.1 --port 8088 --reload
```

**啟動順序（Lifespan）：**
1. AgentRunner.start()
2. MCP Manager 啟動
3. ChannelManager 啟動（每頻道 4 workers，queue 上限 1000）
4. CronManager 啟動（載入 jobs + 註冊 heartbeat）
5. ChatManager 啟動
6. ConfigWatcher 啟動（每 2 秒輪詢 config.json 變更）
7. MCPConfigWatcher 啟動

### 專案進入點

```python
# src/copaw/__init__.py   → 套件初始化
# src/copaw/cli/main.py   → CLI 入口（click group）
# src/copaw/app/_app.py   → FastAPI 應用入口（Lifespan 管理）
```

### 常數與路徑

所有系統常數定義在 `src/copaw/constant.py`：

```python
# 工作目錄路徑
WORKING_DIR = Path(os.environ.get("COPAW_WORKING_DIR", "~/.copaw")).expanduser()

# 各種檔案路徑（相對於 WORKING_DIR）
CONFIG_FILE = WORKING_DIR / os.environ.get("COPAW_CONFIG_FILE", "config.json")
PROVIDERS_FILE = WORKING_DIR / "providers.json"
JOBS_FILE = WORKING_DIR / os.environ.get("COPAW_JOBS_FILE", "jobs.json")
ACTIVE_SKILLS_DIR = WORKING_DIR / "active_skills"
CUSTOMIZED_SKILLS_DIR = WORKING_DIR / "customized_skills"
MEMORY_FILE = WORKING_DIR / "MEMORY.md"
AGENTS_MD_FILE = WORKING_DIR / "AGENTS.md"
SOUL_MD_FILE = WORKING_DIR / "SOUL.md"
```

---

## CLI 命令完整參考

CoPaw 提供完整的 CLI 工具（`copaw` 命令），全域選項為 `--host` 和 `--port`（預設讀取 config.json 或 `127.0.0.1:8088`）。

### copaw app — 啟動服務

```bash
copaw app [--host HOST] [--port PORT] [--reload] [--workers N] [--log-level LEVEL] [--hide-access-paths]
```

### copaw init — 初始化工作目錄

```bash
copaw init [--force] [--defaults] [--accept-security]
```

### copaw channels — 管理頻道

```bash
copaw channels list                     # 列出所有頻道設定
copaw channels config                   # 互動式設定頻道
copaw channels install <key> [--path P] [--url U]  # 安裝自訂頻道
copaw channels add <key> [--path P] [--url U] [--configure/--no-configure]  # 安裝並加入 config
copaw channels remove <key> [--keep-config/--no-keep-config]  # 移除自訂頻道
```

### copaw chats — 管理聊天

```bash
copaw chats list [--user-id ID] [--channel CH] [--base-url URL]  # 列出聊天
copaw chats get <chat_id>               # 取得聊天詳情
copaw chats create [-f FILE] [--name N] [--session-id S] [--user-id U] [--channel C]  # 建立聊天
copaw chats update <chat_id>            # 更新聊天名稱
copaw chats delete <chat_id>            # 刪除聊天
```

### copaw cron — 管理定時任務

```bash
copaw cron list                         # 列出 cron jobs
copaw cron get <job_id>                 # 取得 job 詳情
copaw cron state <job_id>               # 取得 job 狀態
copaw cron create [-f FILE] [--type T] [--name N] [--cron EXPR] [--channel C] \
  [--target-user U] [--target-session S] [--text T] [--timezone TZ] \
  [--enabled/--no-enabled] [--mode M]   # 建立 job
copaw cron delete <job_id>              # 刪除 job
copaw cron pause <job_id>               # 暫停 job
copaw cron resume <job_id>              # 恢復 job
copaw cron run <job_id>                 # 立即執行 job
```

### copaw env — 管理環境變數

```bash
copaw env list                          # 列出環境變數
copaw env set <key> <value>             # 設定環境變數
copaw env delete <key>                  # 刪除環境變數
```

### copaw models — 管理 LLM 與 Provider

```bash
copaw models list                       # 列出 providers 與設定
copaw models config                     # 互動式設定 providers
copaw models config-key [provider_id]   # 設定 API key
copaw models set-llm                    # 設定目前使用的 LLM
copaw models add-provider <id> [-n NAME] [-u URL] [--api-key-prefix P]  # 新增自訂 provider
copaw models remove-provider <id> [-y]  # 移除自訂 provider
copaw models add-model <provider_id> [-m MODEL] [-n NAME]  # 新增模型
copaw models remove-model <provider_id> [-m MODEL]         # 移除模型
copaw models download <repo_id> [-f FILE] [-b BACKEND] [-s SOURCE]  # 下載本地模型
copaw models local [-b BACKEND]         # 列出已下載本地模型
copaw models remove-local <model_id> [-y]  # 刪除本地模型
copaw models ollama-pull <model_name>   # 下載 Ollama 模型
copaw models ollama-list                # 列出 Ollama 模型
copaw models ollama-remove <model_name> [-y]  # 移除 Ollama 模型
```

### copaw skills — 管理 Skills

```bash
copaw skills list                       # 列出 skills 與啟用狀態
copaw skills config                     # 互動式啟用/停用 skills
```

### copaw clean — 清空工作目錄

```bash
copaw clean [--yes] [--dry-run]
```

### copaw uninstall — 解除安裝 CoPaw

```bash
copaw uninstall [--purge] [--yes]
```

---

## 新增 API 端點

### 步驟一：建立 Router

在 `src/copaw/app/routers/` 建立新的 Router 檔案：

```python
# src/copaw/app/routers/my_feature.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

router = APIRouter(prefix="/my-feature", tags=["my-feature"])

class MyFeatureConfig(BaseModel):
    name: str
    value: str

@router.get("/")
async def list_items():
    """列出所有項目"""
    return {"items": []}

@router.post("/")
async def create_item(config: MyFeatureConfig):
    """建立新項目"""
    # 實作邏輯
    return {"id": "new_id", **config.dict()}

@router.put("/{item_id}")
async def update_item(item_id: str, config: MyFeatureConfig):
    """更新項目"""
    # 實作邏輯
    return {"id": item_id, **config.dict()}

@router.delete("/{item_id}")
async def delete_item(item_id: str):
    """刪除項目"""
    # 實作邏輯
    return {"success": True}
```

### 步驟二：註冊到主 Router

在 `src/copaw/app/routers/__init__.py` 中加入：

```python
from .my_feature import router as my_feature_router

# 在 include_router 列表中新增
app.include_router(my_feature_router)
```

### 步驟三：設定對應的 Pydantic 模型

若需要持久化設定，在 `src/copaw/config/config.py` 新增 Pydantic Model：

```python
class MyFeatureConfig(BaseModel):
    name: str = "default_name"
    value: str = ""
    enabled: bool = False
```

---

## 新增頻道（Channel）

### 步驟一：了解 BaseChannel 介面

```python
# src/copaw/app/channels/base.py
class BaseChannel(ABC):
    def build_agent_request_from_native(self, payload) -> AgentRequest:
        """原生 payload → AgentRequest"""
        ...

    async def consume_one(self, payload) -> None:
        """處理單一 payload"""
        ...

    async def send(self, session_id, text) -> None:
        """發送文字訊息"""
        ...

    async def send_content_parts(self, session_id, parts) -> None:
        """發送多種內容（文字、圖片、檔案等）"""
        ...

    async def send_event(self, session_id, event) -> None:
        """發送 Runner Event"""
        ...

    @classmethod
    def from_env(cls) -> "BaseChannel":
        """從環境變數建立"""
        ...

    @classmethod
    def from_config(cls, config) -> "BaseChannel":
        """從設定物件建立"""
        ...
```

**MessageRenderer**（`channels/renderer.py`）：負責將 Agent 回覆轉換為可發送的 parts（文字、圖片、影片、音訊、檔案），處理 tool call、tool output 的渲染。支援 markdown、code fence、emoji 等多種渲染風格。

### 步驟二：實作新頻道

```python
# src/copaw/app/channels/my_channel/channel.py
from ..base import BaseChannel

class MyChannel(BaseChannel):
    def __init__(self, config: MyChannelConfig):
        self.config = config
        self._client = None

    async def start(self) -> None:
        # 初始化客戶端
        self._client = MySDKClient(token=self.config.token)

        # 設定訊息處理器
        @self._client.on_message
        async def handle_message(msg):
            await self._on_message_received(
                user_id=msg.user_id,
                session_id=f"my_channel_{msg.user_id}",
                content=msg.text
            )

        # 啟動監聽
        await self._client.start()

    async def stop(self) -> None:
        if self._client:
            await self._client.close()

    async def send_message(self, session_id, message, attachments=None):
        user_id = session_id.replace("my_channel_", "")
        await self._client.send(user_id=user_id, text=message)
```

### 步驟三：新增頻道設定模型

在 `src/copaw/config/config.py` 新增：

```python
class MyChannelConfig(BaseModel):
    token: str = ""
    enabled: bool = False
```

### 步驟四：在頻道設定中新增

```python
class ChannelConfig(BaseModel):
    # 既有頻道...
    my_channel: MyChannelConfig = Field(default_factory=MyChannelConfig)
```

### 步驟五：在 ChannelRegistry 中註冊

```python
# src/copaw/app/channels/registry.py
from .my_channel.channel import MyChannel

CHANNEL_REGISTRY = {
    # 內建頻道：imessage, discord, dingtalk, feishu, qq, telegram, console
    "my_channel": MyChannel,
}
```

> **自訂頻道（替代方案）**：也可以將頻道放在 `~/.copaw/custom_channels/` 目錄下，ChannelRegistry 會自動載入。這不需要修改源碼，適合發布獨立的頻道套件。使用 `copaw channels install` CLI 命令安裝。

---

## 新增內建工具（Tool）

### 步驟一：建立工具函式

```python
# src/copaw/agents/tools/my_tool.py
from agentscope.service import ServiceResponse, ServiceExecStatus

def my_custom_tool(param1: str, param2: int = 10) -> ServiceResponse:
    """
    執行自訂操作。

    Args:
        param1 (str): 第一個參數的說明
        param2 (int): 第二個參數的說明，預設為 10

    Returns:
        ServiceResponse: 包含操作結果
    """
    try:
        # 執行操作
        result = do_something(param1, param2)
        return ServiceResponse(
            status=ServiceExecStatus.SUCCESS,
            content=result
        )
    except Exception as e:
        return ServiceResponse(
            status=ServiceExecStatus.ERROR,
            content=str(e)
        )
```

### 步驟二：在工具初始化中加入

```python
# src/copaw/agents/tools/__init__.py
from .my_tool import my_custom_tool

# 在 get_builtin_tools() 函式中加入
def get_builtin_tools() -> List:
    return [
        # 既有工具...
        my_custom_tool,
    ]
```

---

## 本地模型支援

### 支援的後端

| 後端 | 安裝方式 | 適用平台 |
|------|---------|---------|
| `llamacpp` | `pip install -e ".[llamacpp]"` | 跨平台 |
| `mlx` | `pip install -e ".[mlx]"` | Apple Silicon（M1/M2/M3/M4） |
| `ollama` | `pip install -e ".[ollama]"` + 安裝 Ollama | 跨平台 |

### 模型下載

```bash
# 從 HuggingFace 下載（需要 pip install -e ".[local]"）
copaw models download --repo "Qwen/Qwen2.5-7B-Instruct-GGUF" --backend llamacpp

# 從 ModelScope 下載
copaw models download --repo "qwen/Qwen2.5-7B-Instruct-GGUF" --source modelscope --backend llamacpp
```

### 模型選擇邏輯

- **llama.cpp**：自動優先選擇 `Q4_K_M` 量化的 `.gguf` 檔案
- **MLX**：需要完整目錄（含 `config.json` 和 `*.safetensors`）

---

## 測試

### 執行所有測試

```bash
pytest tests/ -v
```

### 執行特定測試

```bash
pytest tests/test_react_agent_tool_choice.py -v
```

### 測試覆蓋率

```bash
pytest tests/ --cov=copaw --cov-report=html
# 在瀏覽器開啟 htmlcov/index.html 查看報告
```

### 非同步測試設定

`pyproject.toml` 中已設定 `asyncio_mode = "auto"`，所有 `async` 測試函式自動以非同步模式執行：

```python
# tests/test_example.py
import pytest

@pytest.mark.asyncio  # 可選，因為 asyncio_mode = "auto"
async def test_something():
    result = await some_async_function()
    assert result == expected_value
```

### 目前測試覆蓋範圍

目前只有 1 個測試檔案（`test_react_agent_tool_choice.py`），測試 `tool_choice` 正規化邏輯。**測試覆蓋率極低，歡迎貢獻更多測試！**

---

## 程式碼品質

### Pre-commit Hooks

CoPaw 使用嚴格的 Pre-commit hooks 確保程式碼品質：

```bash
# 手動執行所有 hooks
pre-commit run --all-files

# 執行特定 hook
pre-commit run black --all-files
pre-commit run mypy --all-files
```

### 主要工具設定

| 工具 | 說明 | 設定 |
|------|------|------|
| `black` | Python 程式碼格式化 | line-length=79，跳過 skills/ |
| `flake8` | Python 語法檢查 | 設定在 `.flake8` |
| `mypy` | Python 型別檢查 | 跳過 skills/ |
| `pylint` | Python 靜態分析 | 跳過 skills/，多數規則停用 |
| `prettier` | TypeScript 格式化 | 只處理 `.tsx` 檔案，跳過 console/ |
| `detect-private-key` | 防止提交 API Key | - |

### `.flake8` 設定概覽

```ini
[flake8]
max-line-length = 79
exclude = src/copaw/agents/skills/
```

### 常見 Lint 問題修復

```bash
# 自動修復 black 格式問題
black src/copaw/ --line-length 79 --exclude src/copaw/agents/skills/

# 查看 mypy 型別錯誤
mypy src/copaw/ --exclude src/copaw/agents/skills/
```

---

## CI/CD 流程

### `.github/workflows/pre-commit.yml`

**觸發條件**：所有 push 和 PR

**執行流程：**
1. Ubuntu 環境 + Python 3.10
2. `pip install -e ".[dev]"`
3. `pre-commit run --all-files`（執行所有程式碼品質檢查）

### `.github/workflows/publish-pypi.yml`

**觸發條件**：建立 GitHub Release

**執行流程：**
1. `cd console && npm ci && npm run build`（建置前端）
2. `cp -r console/dist/* src/copaw/console/`（嵌入前端）
3. `python -m build`（打包 Python 套件）
4. 使用 `PYPI_API_TOKEN` 發布到 PyPI

### `.github/workflows/deploy-website.yml`

**觸發條件**：push 到 main 分支且 `website/` 目錄有變更

**執行流程：**
1. pnpm 9 + Node 20 環境
2. `pnpm install --frozen-lockfile`
3. `pnpm run build`（設定 `VITE_BASE_PATH=/`）
4. 部署到 GitHub Pages（CNAME: `copaw.agentscope.io`）

---

## 部署方式

### 1. pip 安裝（推薦個人使用）

```bash
pip install copaw

# 初始化
copaw init --defaults

# 設定 LLM（編輯 ~/.copaw/providers.json 或透過前端）

# 啟動
copaw app  # 啟動於 http://127.0.0.1:8088
```

### 2. Docker（推薦伺服器部署）

```bash
# 使用最新版映像
docker pull agentscope/copaw:latest

# 啟動（綁定 8088 port，掛載工作目錄）
docker run -d \
  --name copaw \
  -p 8088:8088 \
  -v copaw-data:/app/working \
  -e DASHSCOPE_API_KEY=sk-your-key \
  agentscope/copaw:latest
```

**使用自訂工作目錄：**
```bash
docker run -d \
  -p 8088:8088 \
  -v /path/to/your/.copaw:/app/working \
  -e COPAW_WORKING_DIR=/app/working \
  agentscope/copaw:latest
```

### 3. Supervisor（長期運行）

`deploy/` 目錄下有 Supervisor 設定範例：

```ini
[program:copaw]
command=/path/to/venv/bin/copaw app
directory=/home/user
autostart=true
autorestart=true
stderr_logfile=/var/log/copaw/err.log
stdout_logfile=/var/log/copaw/out.log
```

### 4. 一鍵安裝腳本

```bash
# macOS/Linux（免 Python 環境）
curl -fsSL https://copaw.agentscope.io/install.sh | bash

# Windows（PowerShell，免 Python 環境）
irm https://copaw.agentscope.io/install.ps1 | iex
```

---

## 依賴項目說明

### Python 核心依賴

| 套件 | 版本 | 用途 |
|------|------|------|
| `agentscope` | `==1.0.16.dev0` | ReActAgent 核心框架（固定版本）|
| `agentscope-runtime` | `==1.1.0` | Agent 執行期引擎（固定版本）|
| `uvicorn` | `>=0.40.0` | ASGI 伺服器 |
| `apscheduler` | `>=3.11.2,<4` | 定時任務（不使用 v4，API 有破壞性變更）|
| `playwright` | `>=1.49.0` | 瀏覽器自動化（需另執行 `playwright install`）|
| `reme-ai` | `==0.3.0.1` | 長期記憶 RAG（固定版本）|
| `transformers` | `>=4.30.0` | Tokenizer（計算 Token 數）|
| `onnxruntime` | `<1.24` | 本地模型推理 |
| `mss` | `>=9.0.0` | 桌面截圖 |

### 注意事項

1. **agentscope 版本鎖定**：CoPaw 依賴特定版本的 agentscope，升級時需仔細測試
2. **APScheduler v3 vs v4**：系統使用 APScheduler v3（`<4`），v4 有 API 破壞性變更
3. **Playwright 安裝**：`pip install copaw` 後需執行 `playwright install chromium` 安裝瀏覽器
4. **Windows Chroma 相容性**：Windows 上 ChromaDB 可能有相容性問題，建議使用 `MEMORY_STORE_BACKEND=local`

---

## 版本發布流程

### 版本號格式

遵循 PEP 440，當前為 `0.0.5b1`（Beta 版本）：
- `0.0.5b1` → Beta 1
- `0.0.5b2` → Beta 2
- `0.0.5` → 正式版
- `0.0.6` → 下一個版本

### 發布步驟

1. 更新 `src/copaw/__version__.py` 中的版本號
2. 更新 `pyproject.toml` 中的版本號
3. 建立 Git Tag（`v0.0.5`）
4. 在 GitHub 建立 Release
5. GitHub Actions 自動打包並發布到 PyPI

### 本地打包測試

```bash
# 安裝建置工具
pip install build

# 先建置前端
cd console && npm run build && cd ..
cp -r console/dist/* src/copaw/console/

# 打包
python -m build

# 測試安裝（使用 testpypi）
pip install --index-url https://test.pypi.org/simple/ copaw
```

---

## 常見開發問題

### Q: 如何在不重啟服務的情況下測試 Agent 修改？

修改 `AGENTS.md`、`SOUL.md` 或 `PROFILE.md` 後，在聊天介面輸入 `/new` 開始新對話，新對話會載入最新的 Prompt。

### Q: 如何偵錯工具呼叫？

在前端設定 `show_tool_details: true`（預設），可以在聊天介面看到所有工具的輸入/輸出。

### Q: 如何查看詳細日誌？

```bash
export COPAW_LOG_LEVEL=DEBUG
copaw app
```

### Q: 如何新增對 Python 3.14 的支援？

修改 `pyproject.toml` 中的 `requires-python = ">=3.10,<3.14"` 為 `">=3.10,<3.15"`，並測試相容性。

### Q: Skills 修改後需要重啟嗎？

透過 API 啟用/停用 Skills 不需要重啟。但若直接修改 `active_skills/` 目錄，需要開始新的對話 Session 才能載入最新內容。

### Q: 如何查看查詢出錯的詳細資訊？

CoPaw 在查詢出錯時會將 traceback 和 agent state 寫入暫存 JSON 檔案（透過 `query_error_dump.py`），可在暫存目錄中找到對應的 dump 檔案進行分析。

### Q: CoPawAgent 到底註冊了哪些內建工具？

以下工具會在 `_create_toolkit()` 中註冊：

| 工具 | 說明 |
|------|------|
| `execute_shell_command` | 執行 Shell 指令（timeout=60） |
| `read_file` | 讀取檔案（可指定行範圍） |
| `write_file` | 寫入檔案 |
| `edit_file` | 尋找與替換 |
| `browser_use` | Playwright 瀏覽器控制（多 action） |
| `desktop_screenshot` | 桌面截圖 |
| `send_file_to_user` | 傳送檔案給使用者 |
| `get_current_time` | 取得當前時間 |
| `memory_search` | 搜尋長期記憶（需啟用 MemoryManager） |

> **注意**：`append_file`、`grep_search`、`glob_search` 雖然在 `tools/__init__.py` 有匯出，但**不會**被 `_create_toolkit()` 註冊到 Agent。這些工具在 Skills 的腳本中可能被使用。
