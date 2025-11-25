# PROJECT_OVERVIEW - BuildingAI

## 1. 專案目標與 Domain

BuildingAI 是一個**企業級開源智能 Agent 平台**，主要目標與場景（摘自專案 README）：

- 面向：AI 開發者、AI 創業者與企業團隊。
- 透過可視化組態（DIY），建置原生的企業 AI 應用，而非僅嵌入式聊天框。
- 提供內建能力：
    - 智能 Agents（具記憶、目標與工具使用能力）。
    - MCP 整合與工具調用。
    - RAG pipeline 與知識庫管理。
    - 多家大模型聚合與統一 API。
    - 使用者註冊、會員訂閱、計費與營運能力。

平台可透過 Docker 一鍵部署，也支援以 Node.js + pnpm 直接在伺服器上啟動。

---

## 2. 技術棧概要

> 以下資訊來自 `README.md`、根目錄 `package.json`、`pnpm-workspace.yaml`、`.env.example`
> 與部分 Nuxt 設定檔。

### 2.1 Backend（API 與服務）

- **框架與語言**
    - Node.js（`engines.node >= 22`）
    - NestJS 11.x
    - TypeORM 0.3.x
    - TypeScript 5.x
- **資料庫與快取**（依 `.env.example`）
    - Database：PostgreSQL
        - `DB_TYPE=postgres`
        - 預設本機連線：`DB_HOST=localhost`, `DB_PORT=5432`, `DB_DATABASE=buildingai`
    - Redis：作為快取與佇列基礎設施
        - `REDIS_HOST=localhost`, `REDIS_PORT=6379`
- **其它後端相關套件**
    - BullMQ（佇列與背景任務）
    - JWT 驗證（`@nestjs/jwt`）
    - 健康檢查與監控（`@nestjs/terminus` 等）

### 2.2 Frontend（Web / Console）

- **框架與工具**
    - Nuxt 4.x
    - Vue 3.x
    - Pinia 3.x（狀態管理）
    - Nuxt UI、TailwindCSS 4.x
    - Vite 7.x（bundler / dev server）
- **部分前端依賴**（依 `pnpm-workspace.yaml` 的 `web` catalog 與 `package.json`）：
    - UI 與互動：`@nuxt/ui`, `@vueuse/*`, `motion-v`, `reka-ui`
    - 圖表與可視化：`echarts`
    - Rich text 編輯與 markdown：`@tiptap/*`, `tiptap-markdown`, `vue-renderer-markdown`
    - i18n：`@nuxtjs/i18n`, `vue-i18n`

### 2.3 Monorepo 與工具鏈

- 套件管理：pnpm（`packageManager: pnpm@10.20.0`）
- 任務協調：Turborepo（`turbo`）
    - 統一管理 `build`、`dev`、`lint`、`check-types`、`clean` 等任務。
- Process 管理與部署：PM2
- Docker / Docker Compose：官方推薦部署方式，對應專案內的 `docker/` 與 `docker-compose.yml`。

---

## 3. 專案目錄結構（高階）

> 僅列出目前已確認的主要目錄，後續可視需要補充更細的子模組結構。

```text
根目錄 d:/app/BuildingAI
├─ .git/                  # Git 版本控制
├─ assets/                # 專案 banner / 截圖等靜態資源
├─ docker/                # Docker 相關腳本與設定
├─ docs/                  # 規則、文件與 overview 類文檔
│  ├─ rule/               # ELF 規則與 AI Onboarding 文檔
│  ├─ todo/               # 每日 / 每主題 todoYYYY-MM-DD-XX.md
│  └─ PROJECT_OVERVIEW.md # 本檔（專案總覽）
├─ extensions/            # 可安裝的擴充功能（extensions/*）
├─ packages/              # Monorepo 主要程式碼（多個 workspace）
│  ├─ api/                # 後端 API 與服務（NestJS + TypeORM）
│  ├─ core/               # 核心共用模組（domain / infrastructure 等）
│  ├─ cli/                # CLI 工具與安裝 / 運維腳本
│  ├─ web/                # Web 相關專案（前端 UI / Web 套件）
│  └─ @buildingai/        # 共用 npm 套件（如 nuxt presets、stores 等）
├─ public/                # 靜態公開資源
├─ scripts/               # 專案維運腳本（如 sync-env）
├─ storage/               # 執行期儲存或暫存用目錄
├─ templates/             # 模板相關檔案
├─ .env.example           # 環境變數範本
├─ docker-compose.yml     # Docker Compose 定義
├─ package.json           # Monorepo 根設定與 scripts
├─ pnpm-lock.yaml         # pnpm 鎖檔
├─ pnpm-workspace.yaml    # Workspace 與版本 catalog 設定
├─ turbo.json             # Turborepo 任務設定
├─ README.md              # 英文說明
└─ README.zh-CN.md        # 中文說明
```

---

## 4. 核心子專案概觀

> 本節僅描述目前已明確確認的子專案與角色，後續可在對應檔案中（如 FRONTEND_ARCH、API_OVERVIEW）補充細節。

### 4.1 packages/web/buildingai-ui（Nuxt Web 應用）

- 位置：`packages/web/buildingai-ui`
- 角色：BuildingAI 主 Web / Console 前端應用。
- 主要設定（來自 `nuxt.config.ts`）：
    - 使用 `defineBuildingAIConfig`（來自 `@buildingai/nuxt`）作為 Nuxt 設定封裝。
    - dev server 預設埠：`4091`。
    - 別名：
        - `@`、`~` → `./app` 目錄。
    - 全域樣式：`assets/styles/globals.css`。
    - 使用 TailwindCSS Vite 插件、Nuxt Icon 自訂圖示集合等。

### 4.2 packages/web/@buildingai/nuxt（Nuxt 設定與擴充）

- 位置：`packages/web/@buildingai/nuxt`
- 角色：封裝通用 Nuxt 設定與模組，供 `buildingai-ui` 等專案使用。
- 主要特徵（來自 `src/nuxt.config.ts`）：
    - 啟用模組：`@vueuse/nuxt`、`@pinia/nuxt`、`@nuxt/ui`、`@nuxtjs/i18n`、`nuxt-svgo` 等。
    - Vite 設定：
        - `optimizeDeps.include` / `exclude` 針對編輯器、渲染器等套件優化。
        - 透過 `alias` 將 `monaco-editor`、`shiki` 等重定向為空模組，減少 bundle 體積。
    - TypeScript：
        - 嚴格模式（`strict: true`），並透過 `paths` 指向 API 型別定義等共用程式碼。

### 4.3 packages/web/@buildingai/stores（共用 Pinia stores）

- 位置：`packages/web/@buildingai/stores/src`
- 角色：共用的前端狀態管理模組。
- 已確認的 store 範例：
    - `app.ts`：
        - 管理站點設定、登入方式、系統資訊等。
        - 包含系統初始化檢查（決定是否導向 `/install`）。
    - `user.ts`：
        - 管理登入 token、暫存 token、使用者資訊與登入／登出流程。
        - via `@buildingai/service/webapi/user` 呼叫後端 API 取得使用者資料。

### 4.4 packages/api、packages/core、packages/cli（後端與工具）

> 本小節僅標記事件角色，詳細架構與 API 協定建議在 `API_OVERVIEW.md` 或其他後端專用文件中補充。

- `packages/api/`
    - 預期為主要後端 API 服務專案，採用 NestJS + TypeORM 與 PostgreSQL。
- `packages/core/`
    - 預期為領域與共用核心程式碼（domain models、共用 service、infrastructure 抽象等）。
- `packages/cli/`
    - 提供 `buildingai` CLI（對應根目錄 `scripts` / `pm2:*` 指令）。
    - 作為安裝、啟動、升級與進程管理的入口。

---

## 5. 啟動與 build 流程（高階）

> 實際指令請以 `package.json` 與官方文件為準，本節僅總結常用路徑。

### 5.1 使用 Docker Compose（官方推薦）

1. 複製環境變數範本：

    ```bash
    cp .env.example .env
    ```

2. 視環境調整 `.env`（特別是 `APP_DOMAIN`、資料庫與 Redis 設定）。
3. 啟動服務：

    ```bash
    docker compose up -d
    ```

4. 服務完成啟動後，瀏覽器開啟：
    - 初始安裝流程：`http://localhost:4090/install`

### 5.2 使用 Node.js + pnpm（本機開發）

- 根目錄 `package.json` 主要 scripts：
    - `pnpm dev`

        ```bash
        pnpm dev
        ```

        - 透過 Turbo 執行各子專案的 `dev` 任務（具體包含哪些子專案，需再視各 packages 的
          `package.json` 而定）。

    - `pnpm build`

        ```bash
        pnpm build
        ```

        - 透過 Turbo 執行 monorepo 的 `build` 任務（包含 API / Web / CLI 等）。

    - `pnpm docker:up` / `pnpm docker:down`

        ```bash
        pnpm docker:up   # 相當於 docker compose up -d
        pnpm docker:down # 相當於 docker compose down
        ```

- 開發時常見連線端口：
    - API / Web 入口（經 Node 伺服器與反向代理）：`4090`
    - Nuxt 開發伺服器（`buildingai-ui`）：`4091`（依 `nuxt.config.ts`）。

---

## 6. 環境變數與部署考量（簡要）

> 詳細說明建議另開部署文件，本節僅標出與本專案理解高度相關的 key。

- **Base**
    - `APP_NAME`：專案名稱（預設 `BuildingAI`）。
    - `APP_DOMAIN`：部署到正式環境時應設定為實際網域。
- **Server**
    - `SERVER_PORT`：預設 `4090`。
    - `SERVER_CORS_*`、`SERVER_SHOW_DETAILED_ERRORS`、`SERVER_IS_DEMO_ENV` 等運行行為控制。
- **Database（PostgreSQL）**
    - `DB_TYPE`, `DB_HOST`, `DB_PORT`, `DB_USERNAME`, `DB_PASSWORD`, `DB_DATABASE`。
    - `DB_SYNCHRONIZE` / `DB_DEV_SYNCHRONIZE` 控制 schema 同步行為。
- **Redis / Cache**
    - `REDIS_HOST`, `REDIS_PORT`, `REDIS_DB`, `REDIS_TTL`。
    - `CACHE_TTL`, `CACHE_MAX_ITEMS`。
- **Web 前端相關**
    - `VITE_DEVELOP_APP_BASE_URL`：預設 `http://localhost:4090`。
    - `VITE_PRODUCTION_APP_BASE_URL`：正式環境 base URL。
    - `VITE_APP_WEB_API_PREFIX` / `VITE_APP_CONSOLE_API_PREFIX`：前端呼叫 API 時的路徑前綴（預設
      `/api` 與 `/consoleapi`）。

---

## 7. 後續文件與延伸

依 `ELF_AI_ONBOARDING.md` 建議，後續可由本檔延伸出：

- `docs/FRONTEND_ARCH.md`
    - 更詳細說明 Nuxt 應用的啟動流程、路由結構、狀態管理與全域 UI 元件。
- `docs/API_OVERVIEW.md`
    - 描述 API 模組（如 `system`、`systemData`、`onlineDev` 等）與 `src/api/**`、`src/views/**`
      的對應關係。
- `docs/ui-flow-<feature>.md`
    - 僅針對少數需要深入理解的核心功能（如複雜設計器、授權流程、特定 refactor 主題）撰寫 UI/Data
      Flow 文件。

本檔的角色是：**在單一檔案中提供 BuildingAI 專案的整體輪廓**，讓任何人或 AI 可以快速了解：

- 這個專案在做什麼。
- 使用了哪些主要技術與基礎設施。
- 原始碼大致如何分布在 monorepo 中。
- 要從哪裡開始啟動與部署這個系統。
