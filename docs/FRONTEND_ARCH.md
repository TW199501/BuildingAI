# FRONTEND_ARCH - BuildingAI Nuxt Web 應用

## 1. 角色與定位

- 前端主應用專案路徑：`packages/web/buildingai-ui`
- 採用 Nuxt 4 + Vue 3 + Pinia + Nuxt UI + TailwindCSS，透過 `@buildingai/nuxt` 提供的 `defineBuildingAIConfig` 封裝共用設定。
- 主要用途：
  - 承載 BuildingAI 的 Web Portal / Console 介面。
  - 透過動態路由與權限控制呈現各種後台頁面與擴充功能。

---

## 2. 啟動流程（高階）

> 依目前閱讀到的檔案：`packages/web/buildingai-ui/nuxt.config.ts`、`app/app.vue`。

### 2.1 Nuxt 設定入口：`nuxt.config.ts`

- 使用 `defineBuildingAIConfig` 封裝 Nuxt 設定：
  - `ssr` 由 `NUXT_BUILD_SSR` 決定。
  - `pages.pattern: ["!console/**/*.*"]`：
    - 排除 `console` 目錄內的檔案由預設 pages router 掃描，改由自訂邏輯（見後續 console 動態路由）。
  - `components`：集中註冊 `app/components` 目錄為局部元件（不使用路徑前綴）。
  - `app.head`：設定 viewport 與 theme-color 等全域 `<head>` meta。
  - `css`：載入 `assets/styles/globals.css` 作為全域樣式來源。
  - `alias`：
    - `@`、`~` → `./app` 目錄（即 Nuxt 應用的根）。
  - `devServer.port = 4091`：開發時前端伺服器埠號。

### 2.2 App 根組件：`app/app.vue`

- 採用 `<script setup lang="ts">`，在最高層：
  - 匯入：
    - `useAppStore()`：來自共用 Pinia store `@buildingai/stores/app`。
    - `useI18n()`、`useColorMode()`、`useAppConfig()` 以及 Nuxt UI 相關設定。
    - `apiRecordAnalyse` 與 `AnalyseActionType.PAGE_VISIT` 進行行為記錄。
  - 啟動時執行的核心流程：
    1. **載入全域站點設定與登入設定**
       - 透過 `useAsyncData("config", () => appStore.getConfig())` 取得站點設定（名稱、LOGO 等）。
       - 透過 `useAsyncData("loginSettings", () => appStore.getLoginSettings())` 取得登入設定。
    2. **設定 `<head>` 與 SEO**
       - 使用 `useHead`：
         - 設定 `<title>` 為 `siteConfig.webinfo.name`。
         - 設定 `meta viewport`、`theme-color`、favicon 與 Apple Touch Icon 等。
         - 以 cookie 或 `locale` 決定 `<html lang>`。
       - 使用 `useSeoMeta`：
         - 設定 description、OG 標籤（標題、描述、圖片、網站名稱等）。
    4. **記錄頁面訪問行為**
       - 取得當前路由 `route.path`，呼叫 `apiRecordAnalyse`：
         - `actionType: PAGE_VISIT`。
         - `extraData` 包含 referrer 與 timestamp。
    3. **注入 UI 主題相關 CSS 變數**
       - 根據 `appConfig.theme.radius` 與 `appConfig.theme.blackAsPrimary` 等設定：
         - 寫入 `--ui-radius`、`--ui-primary` 等 CSS 變數，影響 Nuxt UI 元件外觀。

> 總結：`app.vue` 扮演「載入全域設定 + 設定 Head / SEO + 注入主題樣式 + 記錄訪問」的角色，實際畫面結構則交由 pages / layouts / router 決定。

---

## 3. 路由架構

> 依目前閱讀到的檔案：`app/middleware/route.global.ts`、`app/layouts/console.vue`、`app/utils/menu-helper.ts`，以及目錄結構 `app/pages/**`。

### 3.1 Nuxt pages-based 路由概念

- 應用根目錄：`app/`
  - `pages/`：一般頁面（登入、安裝向導、公開頁等）與 console 頁面（`pages/console/**`）。
  - `layouts/`：定義全域或區域佈局（例如 `console.vue`）。
  - `middleware/`：放置全域或具名路由 middleware（例如 `route.global.ts`）。
- 其中 `pages/console/**` 不直接由 Nuxt 預設 router 掃描（在 `nuxt.config.ts` 中已排除 `console/**/*`），而是透過動態 route 註冊的方式掛載（見 3.3）。

### 3.2 全域路由 Middleware：`app/middleware/route.global.ts`

此檔透過 `defineNuxtRouteMiddleware` 定義了一個**全域路由中介層**，每次路由切換都會走過這裡，主要負責：

1. **處理擴充路由：`/extensions/...`**

   - 檢查目標路徑是否以 `ROUTES.EXTENSIONS` 開頭。
   - 從 `useRuntimeConfig().public.extensions` 取得可用 extension 名稱。
   - 若 extension 存在 → 交由 Nuxt 正常渲染。
   - 若 extension 不存在 → 丟出 `404 Extension not found`。
2. **系統初始化檢查**

   - 使用 `useAppStore().checkSystemInitialization(to.path)`：
     - 系統尚未初始化且目標不是 `/install` → 轉導到 `/install`。
     - 系統已初始化但仍訪問 `/install` → 轉回首頁 `/`。
     - 若取得系統資訊失敗 → 丟出 `404 System not Connected`（fatal）。
3. **登入與導向控制（handleAuth）**

   - 使用 `useUserStore()` 與 `usePermissionStore()`：
     - 若目標路徑屬於 `ROUTES.CONSOLE`：
       - 將 `to.meta.layout` 設為 `console`，並標記 `to.meta.auth = true`（需要驗證）。
     - 若已登入但尚未取得 `userInfo` → 呼叫 `userStore.getUser()` 補全使用者資訊。
     - 若未登入且目標需要驗證（`meta.auth !== false` 且不是登入頁）：
       - 切換 layout 為 `full-screen`。
       - 導向登入頁：`ROUTES.LOGIN?redirect=${to.fullPath}`。
     - 若已登入卻前往登入頁 → 依情況導回上一頁或首頁。
     - 若已登入進入其他頁面 → 呼叫 `userStore.refreshToken()` 延長 token。
4. **Console 區域動態路由與權限檢查**

   - 事前判斷：若目前已載入 menus 且 `to.fullPath` 已能在 router 裡匹配到 console route → 直接放行。
   - 否則：
     1. 呼叫 `permissionStore.loadPermissions()` 取得目前使用者可用的 menu 與 permission。
     2. 非 root（`isRoot === 0`）且 permission 清單為空 → 導向 `/403`。
     3. 以 `buildRoutes(...)` 建立 console routes：
        - `modules = import.meta.glob(["@/pages/console/**/*.vue"], { eager: false })`。
        - 將 menu 設定與對應的 Vue component 映射成 `RouteRecordRaw[]`。
        - 逐一呼叫 `router.addRoute(route)` 註冊進 Nuxt router。
     4. 權限碼檢查：
        - 若目標路由帶有 `meta.permissionCode` 且使用者不是 root，則透過 `permissionStore.hasPermission` 檢查；失敗則導向 `/403`。
     5. Console 首頁導向：
        - 若目標是 `ROUTES.CONSOLE`（或其尾隨 `/` 版本），則導向第一個可用的 routes[0] 或 HOME。
     6. 註冊後再次檢查是否能匹配到 console route：
        - 若仍無對應 → 設定 layout 為 `full-screen` 並導向 `/not-found`。

> 總結：`route.global.ts` 是前端路由層的「中樞」，串起：
>
> - 系統是否已完成安裝。
> - 使用者是否登入與 token 有效期。
> - 是否有訪問該 console 頁面的權限。
> - 如何將後端給的 menu 組態，動態轉換成實際前端路由。

### 3.3 Console Layout：`app/layouts/console.vue`

- 用於所有 console 區域頁面（由 `route.global.ts` 把 layout 指定為 `"console"`）。
- 主要元件與行為：
  - 匯入 `SidebarLayout`、`MixtureLayout`、`SearchModal` 等 console UI 元件。
  - 使用 `useMediaQuery("(max-width: 768px)")` 判斷是否為手機寬度：
    - 手機寬時：
      - 暫存原本的 `consoleLayoutMode`，並將 layout 切換為 `"side"`。
      - 更新 Cookie `LAYOUT_MODE` 方便下次進站沿用。
    - 回到桌機寬時：
      - 還原為原本的 layout 模式，並清空暫存狀態。
  - Template 中：
    - `controls.consoleLayoutMode === 'side'` → 顯示 `<SidebarLayout />`。
    - `controls.consoleLayoutMode === 'mixture'` → 顯示 `<MixtureLayout />`。
    - 無論如何都會渲染 `<SearchModal />` 以提供搜尋能力。

---

## 4. 狀態管理（Pinia）

> 依目前閱讀到的檔案：`packages/web/@buildingai/stores/src/app.ts`、`user.ts`，以及在 `route.global.ts` 中看到的 stores 使用方式。

### 4.1 Store 來源與註冊方式

- 共用 stores 定義在獨立套件：`packages/web/@buildingai/stores/src`：
  - `app.ts` → `useAppStore`
  - `user.ts` → `useUserStore`
  - `permission.ts`（尚未細讀，但在 `route.global.ts` 中有 `usePermissionStore` 的使用）
- 在 `@buildingai/nuxt` 提供的 Nuxt 設定中已啟用 `@pinia/nuxt`，並由各 store 模組內部建立 Pinia 實例並導出對應的 `useXXXStore` 函式供組件呼叫。

### 4.2 App Store（`app.ts`）職責概要

- 全域站點設定：
  - 透過 `apiGetSiteConfig()` 取得網站名稱、Logo、描述等，快取在 `siteConfig` 中。
- 登入設定：
  - 透過 `apiGetLoginSettings()` 取得登入相關設定（登入協議、預設登入方式等）。
- 系統資訊：
  - 透過 `apiGetSystemInfo()` 取得是否已初始化等狀態，供 `checkSystemInitialization()` 使用。
- 系統初始化檢查：
  - `checkSystemInitialization(path: string)`：
    - 判斷是否需要導向 `/install` 或首頁 `/`，在 `route.global.ts` 中扮演第一道防線。

### 4.3 User Store（`user.ts`）職責概要

- Token 與登入狀態：
  - 管理 `token`、`temporaryToken`、`loginTimeStamp` 等狀態，並與 cookie 同步。
  - `isLogin`：判斷目前是否為登入狀態。
- 登入與登出流程：
  - `login(newToken)`：
    - 設定 token 與登入時間戳，呼叫 `getUser()` 取得使用者資訊。
    - 根據 URL `redirect` 參數決定導向首頁或指定路徑。
  - `logout(expire?)`：
    - 清除 token、使用者資訊、PM2 相關本地狀態，並觸發重新載入或導向登入頁。
- 權杖更新：
  - `refreshToken()`：在 token 存在且登入時間戳有效時，每隔一段時間延長 cookie 的有效期。

### 4.4 Permission Store（推測職責）

> 目前尚未直接閱讀 `permission` store 檔案，但從 `route.global.ts` 與 `menu-helper.ts` 的使用方式可推得：

- 持有：
  - `menus`：後端回傳的 menu tree，用於生成 console routes。
  - `permissions`：使用者擁有的權限碼清單。
- 提供：
  - `loadPermissions()`：向後端取得 menus 與 permissions。
  - `hasPermission(code: string)`：判斷是否擁有某個權限碼。

---

## 5. 後續待補區塊（TODO）

> 以下內容會在之後進一步閱讀 `app/pages/**`、`app/layouts/**` 以及相關模組後補上。

- **5.1 路由模組分區一覽（初步）**

  - 依目前 `app/pages/` 結構，主要區塊如下（僅列出高階分區，細節待後續補充）：
    - `index.vue`
      - 對應根路由 `/`，作為入口頁面（首頁或主入口 Dashboard）。
    - `403.vue`
      - 無權限時顯示的錯誤頁，與權限檢查（例如 `route.global.ts` 中導向 `/403`）對應。
    - `[...all].vue`
      - Nuxt 的 catch-all route，用於處理未匹配的路由（Not Found 類頁面）。
    - `install/`
      - 安裝精靈與系統初始化流程相關頁面，對應 `/install` 路由與 `checkSystemInitialization()` 的導向邏輯。
    - `login/`
      - 登入／註冊／驗證碼等認證相關頁面，供未登入使用者導向（`ROUTES.LOGIN`）。
    - `console/`
      - Console 後台主功能區的頁面集合，實際路由由 `route.global.ts` + menus 動態掛載到 `pages/console/**` 對應的 `.vue` 檔。
    - `chat/`
      - 與對話／聊天場景相關的頁面（如對話界面、多模型對話等）。
    - `profile/`
      - 使用者個人中心與帳戶設定相關頁面。
    - `public/`
      - 對外公開頁或不需登入即可存取的頁面（landing、小工具或分享頁等）。
    - `agreement/`
      - 服務條款、隱私權政策等協議類頁面。
    - `micropage/`
      - 可能用於對外分享或嵌入的「微頁面」，具體行為待後續補充。
    - `webview/`
      - 可能作為在 iframe / WebView 中嵌入顯示內容的容器頁，具體用途待後續補充。
- **5.2 Middleware / Plugins 與 Nuxt 模組間的關係**

  - 說明 `app/middleware` 其他檔案（若有）與 Nuxt modules（在 `modules/` 目錄中）的互動方式。
- **5.3 Extensions 前端路由結構**

  - 依 `ROUTES.EXTENSIONS` 與 `EXTENSION_API_URL` 等設定，補充 extensions 的路由與部署方式。

本檔目前作為 **FRONTEND_ARCH 初版骨架**，目的是：

- 把已知的 Nuxt 啟動流程、路由中樞與狀態管理邏輯先固定下來。
- 之後在閱讀更細的 pages 結構與功能模組時，可以就近在本檔補充，而不需要重新發明結構。
