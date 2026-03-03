# 前端開發指南

CoPaw 前端位於 `console/` 目錄，使用 React 18 + TypeScript + Ant Design 5 構建，透過 Vite 6 打包後嵌入到 FastAPI 後端靜態服務。

---

## 目錄

- [技術棧](#技術棧)
- [專案結構](#專案結構)
- [頁面路由](#頁面路由)
- [主要頁面說明](#主要頁面說明)
- [UI 元件系統](#ui-元件系統)
- [國際化（i18n）](#國際化i18n)
- [API 通訊](#api-通訊)
- [本地開發](#本地開發)
- [建置與部署](#建置與部署)

---

## 技術棧

| 技術 | 版本 | 說明 |
|------|------|------|
| React | ^18 | UI 框架 |
| TypeScript | ~5.8.3 | 型別系統 |
| Ant Design | ^5.29.1 | UI 元件庫 |
| antd-style | ^3.7.1 | CSS-in-JS（Token 系統） |
| @agentscope-ai/chat | ^1.1.50 | AgentScope 聊天元件 |
| @agentscope-ai/design | ^1.0.14 | AgentScope 設計系統 |
| @agentscope-ai/icons | ^1.0.46 | AgentScope 圖示庫 |
| @ant-design/x-markdown | ^2.2.2 | Markdown 渲染 |
| ahooks | ^3.9.6 | React Hooks 工具庫 |
| react-router-dom | ^7.13.0 | 前端路由 |
| react-i18next | ^16.5.4 | 國際化 |
| i18next | ^25.8.4 | i18n 引擎 |
| lucide-react | ^0.562.0 | 圖示元件 |
| Vite | ^6.3.5 | 建置工具 |
| Less | ^4.5.1 | CSS 前處理器 |

---

## 專案結構

```
console/
├── package.json
├── vite.config.ts
├── tsconfig.json
├── index.html
└── src/
    ├── main.tsx              # 應用入口
    ├── App.tsx               # 根元件
    ├── layouts/
    │   └── MainLayout/       # 主版面佈局
    │       ├── index.tsx     # 路由定義
    │       └── Sidebar.tsx   # 側欄導航
    │
    ├── pages/                # 頁面元件
    │   ├── Chat/             # 聊天頁面
    │   ├── Control/          # 控制面板頁面
    │   │   ├── Channels/     # 頻道設定
    │   │   ├── Sessions/     # 會話管理
    │   │   ├── CronJobs/     # 定時任務
    │   │   └── Heartbeat/    # 心跳設定
    │   ├── Agent/            # Agent 設定頁面
    │   │   ├── Workspace/    # 工作目錄
    │   │   ├── Skills/       # Skills 管理
    │   │   ├── MCP/          # MCP 客戶端
    │   │   └── Config/       # Agent 參數
    │   └── Settings/         # 系統設定頁面
    │       ├── Models/       # LLM 模型管理
    │       └── Environments/ # 環境變數
    │
    ├── components/           # 共用元件
    │   ├── Layout/           # 整體版面（側欄、頂欄）
    │   └── ...
    │
    ├── hooks/                # 自訂 React Hooks
    │   ├── useApi.ts         # API 請求封裝
    │   └── ...
    │
    ├── services/             # API 服務層
    │   ├── models.ts         # /models API
    │   ├── config.ts         # /config API
    │   ├── cron.ts           # /cron API
    │   ├── skills.ts         # /skills API
    │   ├── mcp.ts            # /mcp API
    │   └── envs.ts           # /envs API
    │
    ├── types/                # TypeScript 型別定義
    │   └── index.ts
    │
    ├── i18n/                 # 國際化資源
    │   ├── en/               # 英文翻譯
    │   └── zh/               # 中文翻譯
    │
    └── styles/               # 全域樣式
        └── global.less
```

---

## 頁面路由

路由定義在 `console/src/layouts/MainLayout/index.tsx`，使用 React Router v7。

| 路徑 | 頁面元件 | 說明 |
|------|---------|------|
| `/` | （重新導向至 `/chat`） | 預設頁面 |
| `/chat` | `pages/Chat` | 主要聊天介面 |
| `/channels` | `pages/Control/Channels` | 頻道設定管理 |
| `/sessions` | `pages/Control/Sessions` | 會話列表與管理 |
| `/cron-jobs` | `pages/Control/CronJobs` | 定時任務 CRUD |
| `/heartbeat` | `pages/Control/Heartbeat` | 心跳功能設定 |
| `/workspace` | `pages/Agent/Workspace` | 工作目錄檔案瀏覽 |
| `/skills` | `pages/Agent/Skills` | Skills 安裝/啟用/停用 |
| `/mcp` | `pages/Agent/MCP` | MCP 客戶端管理 |
| `/agent-config` | `pages/Agent/Config` | Agent 行為參數設定 |
| `/models` | `pages/Settings/Models` | LLM Provider 與模型管理 |
| `/environments` | `pages/Settings/Environments` | 環境變數管理 |

---

## 主要頁面說明

### Chat（聊天介面）

主要的對話介面，核心功能：
- 即時聊天（使用 AgentScope Chat 元件）
- 顯示/隱藏工具呼叫細節
- Session 切換（左側 Session 列表）
- 訊息 Markdown 渲染（@ant-design/x-markdown）
- 檔案上傳與傳送

**使用的關鍵元件：**
- `@agentscope-ai/chat`：聊天泡泡、輸入框元件
- `@agentscope-ai/design`：bailianTheme 主題

### Control/CronJobs（定時任務）

Cron 排程管理介面：
- 任務列表（顯示名稱、排程、狀態）
- 建立新任務（Cron 表達式輸入 + 任務描述）
- 暫停/恢復/立即執行任務
- 刪除任務

### Agent/Skills（Skills 管理）

Skills 的完整管理介面：
- 已啟用 Skills 列表
- 內建 Skills（可啟用/停用）
- 自訂 Skills（可建立/刪除）
- Skills Hub 搜尋與安裝

### Settings/Models（模型管理）

LLM Provider 設定介面：
- Provider 列表（含連線狀態指示）
- API Key 輸入（遮罩顯示）
- 新增/移除模型
- 測試連線
- 選擇啟用的模型

---

## UI 元件系統

### 主題設定

CoPaw 使用 AgentScope 的 `bailianTheme` 作為基礎設計主題，搭配 Ant Design 5 的 Token 系統：

```tsx
import { bailianTheme } from '@agentscope-ai/design';
import { ConfigProvider } from 'antd';

function App() {
  return (
    <ConfigProvider theme={bailianTheme}>
      {/* ... */}
    </ConfigProvider>
  );
}
```

### 樣式方案

使用 `antd-style` 的 `createStyles` 進行 CSS-in-JS：

```tsx
import { createStyles } from 'antd-style';

const useStyles = createStyles(({ token, css }) => ({
  container: css`
    background: ${token.colorBgContainer};
    border-radius: ${token.borderRadius}px;
    padding: ${token.paddingMD}px;
  `,
}));

function MyComponent() {
  const { styles } = useStyles();
  return <div className={styles.container}>...</div>;
}
```

### 圖示使用

```tsx
// AgentScope 圖示
import { SomeIcon } from '@agentscope-ai/icons';

// Lucide 圖示
import { Settings, MessageCircle } from 'lucide-react';

// Ant Design 圖示
import { PlusOutlined, DeleteOutlined } from '@ant-design/icons';
```

---

## 國際化（i18n）

### 設定

使用 `react-i18next` + `i18next`，支援中文（`zh`）和英文（`en`）。

### 翻譯檔案結構

```
src/i18n/
├── en/
│   ├── common.json    # 通用詞彙
│   ├── chat.json      # 聊天頁面
│   ├── settings.json  # 設定頁面
│   └── ...
└── zh/
    ├── common.json
    ├── chat.json
    ├── settings.json
    └── ...
```

### 使用方式

```tsx
import { useTranslation } from 'react-i18next';

function MyComponent() {
  const { t } = useTranslation('common');
  return <button>{t('save')}</button>;
}
```

### 切換語言

語言跟隨 `config.json` 中的 `agents.language` 設定（`zh` 或 `en`）。

---

## API 通訊

### 基礎 URL

前端透過相對路徑（同源）呼叫後端 API：

```typescript
const API_BASE = '/api';  // 或依環境設定
```

### 使用 ahooks 的 `useRequest`

```tsx
import { useRequest } from 'ahooks';

function SkillsList() {
  const { data, loading, error, refresh } = useRequest(
    () => fetch('/skills').then(res => res.json())
  );

  if (loading) return <Spin />;
  if (error) return <Alert message={error.message} type="error" />;

  return (
    <List dataSource={data} renderItem={item => (
      <List.Item>{item.name}</List.Item>
    )} />
  );
}
```

### 常用 API 呼叫範例

```typescript
// 啟用 Skill
async function enableSkill(skillName: string) {
  const response = await fetch(`/skills/${skillName}/enable`, {
    method: 'POST',
  });
  if (!response.ok) throw new Error('Failed to enable skill');
}

// 建立 Cron Job
async function createCronJob(spec: CronJobSpec) {
  const response = await fetch('/cron/jobs', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(spec),
  });
  return response.json();
}

// 更新環境變數
async function updateEnvs(envs: Record<string, string>) {
  await fetch('/envs', {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(envs),
  });
}
```

---

## 本地開發

### 前置需求

- Node.js 20+
- npm 或 pnpm

### 啟動開發伺服器

```bash
cd console
npm install

# 開發模式（熱重載）
npm run dev
```

預設開發伺服器運行於 `http://localhost:5173`。

### 代理設定（Vite）

為了在開發時呼叫本地後端 API，`vite.config.ts` 設定了代理：

```typescript
// vite.config.ts（推測）
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://127.0.0.1:8088',
        changeOrigin: true,
      },
    },
  },
});
```

### 確保後端已啟動

```bash
# 在另一個終端啟動後端
cd ..
pip install -e ".[dev]"
copaw app
```

---

## 建置與部署

### 建置前端

```bash
cd console
npm run build
```

建置結果輸出到 `console/dist/`。

### 嵌入後端

CI/CD 流程（`.github/workflows/publish-pypi.yml`）會自動：

1. `npm run build` 建置前端
2. 將 `console/dist/*` 複製到 `src/copaw/console/`
3. 後端以 FastAPI 的 `StaticFiles` 提供前端靜態文件服務

**手動執行：**

```bash
cd console
npm run build
cp -r dist/* ../src/copaw/console/
```

### 型別檢查

```bash
cd console
npx tsc --noEmit
```

### Lint 檢查

```bash
cd console
npx prettier --check "src/**/*.tsx"
```

---

## 新增頁面指南

### 步驟一：建立頁面元件

```bash
mkdir console/src/pages/MyPage
touch console/src/pages/MyPage/index.tsx
```

```tsx
// console/src/pages/MyPage/index.tsx
import React from 'react';
import { Card, Typography } from 'antd';
import { createStyles } from 'antd-style';

const useStyles = createStyles(({ token, css }) => ({
  container: css`
    padding: ${token.paddingLG}px;
  `,
}));

const MyPage: React.FC = () => {
  const { styles } = useStyles();

  return (
    <div className={styles.container}>
      <Typography.Title level={2}>My Page</Typography.Title>
      <Card>
        {/* 頁面內容 */}
      </Card>
    </div>
  );
};

export default MyPage;
```

### 步驟二：新增路由

在 `console/src/layouts/MainLayout/index.tsx` 中新增路由：

```tsx
import MyPage from '../../pages/MyPage';

// 在路由設定中新增：
{
  path: '/my-page',
  element: <MyPage />,
}
```

### 步驟三：新增側欄連結

在 `console/src/layouts/MainLayout/Sidebar.tsx` 的側欄導航中新增項目，指定所屬的分組（Chat / Control / Agent / Settings）。

### 步驟四：新增翻譯

在 `console/src/i18n/zh/` 和 `console/src/i18n/en/` 中新增對應翻譯。
