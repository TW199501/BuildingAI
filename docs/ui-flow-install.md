# UI Flow - Install 安裝精靈流程（草稿）

> 狀態：已完成前端 install 頁面第一輪閱讀，以下內容根據 `app/pages/install`
> 及相關 composable 整理；與後端 API 更細欄位對應仍待後續若有需要再補充。

## 1. 流程目的與範圍

- 說明安裝精靈（`/install` 相關頁面）的整體目的：
  - 例：首次啟動系統時，引導管理者完成必要設定，使系統進入可用狀態。
- 範圍：
  - 前端：`packages/web/buildingai-ui/app/pages/install/**`。
  - 與前端路由／狀態相關：
    - `app/middleware/route.global.ts` 中與 `/install` / 系統初始化檢查有關的邏輯。
    - `@buildingai/stores/app` 中 `checkSystemInitialization()` 或其他相關函式。
  - 後端：（僅在需要時補充）與安裝／初始化相關的 API endpoint。

## 2. 路由與畫面流

> 目標：從「系統尚未初始化」到「安裝完成可以正常使用」的完整頁面流。

- **入口路由與 layout**
  - 當系統尚未初始化時，`appStore.checkSystemInitialization(path)` 在 `route.global.ts`
      中會將使用者從任意路徑導向 `/install`。
  - `app/pages/install/index.vue` 透過 `definePageMeta({ layout: "full-screen", auth: false })`：
    - 使用 `full-screen` 佈局。
    - `auth: false`，不需登入即可進入安裝流程。

- **Step 0：歡迎頁（Step0）**
  - 當 `currentStep === 0` 時顯示，為安裝精靈的歡迎畫面：
    - 動態標題動畫：使用 `TrueFocus` 顯示 `install.welcome.title` 文案。
    - 說明文字：`install.welcome.subtitle`。
    - 行為按鈕：
      - 「查看文件」：`<UButton>` 連結
              `https://www.buildingai.cc/docs/introduction/start`（新視窗開啟）。
      - 「開始使用」：點擊後觸發 `@next` 事件，由父層 `index.vue` 的 `handleWelcomeNext` 將
              `currentStep` 設為 `1` 進入下一步。
    - 語言切換：畫面右上角的 `<USelect>` 綁定 `locale`（`LanguageCode`），資料源為
          `languageOptions`；切換後會呼叫 `setLocale` 並更新 `nuxt-ui-language` cookie，使後續頁面沿用
          相同語系。Nuxt `defaultLocale` 已在 `packages/web/@buildingai/nuxt/src/nuxt.config.ts` 改為
          `"zh-TW"`，因此第一次開啟安裝畫面就會顯示繁體中文；若使用者改選其它語言，cookie 將覆蓋
          預設值並套用至整個流程。

- **Step 1：管理員帳號設定（Step1）**
  - 當 `currentStep === 1` 顯示，主要功能：
    - 建立初始管理員帳號（username、password）與可選的個人資訊（avatar、nickname、email 等）。
    - 提供密碼強度檢查與視覺化提示。
    - 要求使用者勾選「同意隱私／使用協議」後才可繼續。
  - 互動流程：
    - 使用 `UForm` + `yup` schema 進行前端驗證：
      - `username`：必填、長度與格式限制（僅英數與底線）。
      - `password`：必填、最小長度限制。
      - `email`：若填寫則必須符合 email 格式。
      - `agreedToTerms`：必須為 `true`。
    - 表單提交：
      - `Step1` 內部驗證成功且 `agreedToTerms` 為 true 時，emit `submit` 事件。
      - 父層 `index.vue` 的 `handleAdminSubmit()` 呼叫 `useInstall().submitAdminInfo()`：
        - 目前實作僅模擬 loading（300ms 延遲）後回傳 `true`，不直接呼叫後端 API。
        - 若回傳 `true`，將 `currentStep` 設為 `2`，進入網站設定步驟。

- **Step 2：網站資訊與品牌設定（Step2）**
  - 當 `currentStep === 2` 顯示，主要功能：
    - 設定網站名稱、描述、Logo 與 Icon 等品牌資訊。
    - 為後續前端顯示（標題、favicon、logo 等）提供資料來源。
  - 互動流程：
    - `UForm` + `yup` schema：
      - `name`（網站名稱）為必填欄位。
    - 上傳 Logo / Icon：
      - 使用 `BdUploader` 搭配 `apiUploadInitFile` 處理檔案上傳（限制副檔名與單檔上傳）。
    - 操作按鈕：
      - 「上一步」：emit `prev` 事件，父層 `handlePrevStep()` 將 `currentStep` 重設為 `1`。
      - 「下一步」：提交表單成功後 emit `submit` 事件，父層 `handleWebsiteSubmit()` 呼叫
              `submitWebsiteConfig()`：
        - 若回傳 `true`，將 `currentStep` 設為 `3`，進入完成頁面。

- **Step 3：安裝完成頁（Step3）**
  - 當 `currentStep === 3` 顯示，主要功能：
    - 告知使用者安裝已完成（`install.completeTitle` / `install.completeDescription`）。
    - 提供兩個導向選項：
      - 「前往首頁」：emit `goHome` → `useInstall().goToHome()` → `router.push("/")`。
      - 「前往控制台」：emit `goConsole` → `useInstall().goToConsole()` →
              `router.push("/console")`。

- **結束條件與後續行為（概念層面）**
  - 安裝成功後，`apiInitializeSystem` 回傳 `isInitialized = true`，`appStore.getSystemInfo()`
      應更新系統狀態為已初始化。
  - 之後 `appStore.checkSystemInitialization()` 再被呼叫時：
    - 對 `/install` 路徑會轉回首頁 `/`。
    - 對其他路徑則不再強制導向安裝流程。

## 3. 狀態與資料流

> 目標：整理安裝流程中前端收集／管理的狀態，以及與後端交換的主要資料結構。

- **本地狀態與表單**
  - 在 `useInstall()` 中：
    - `adminForm: SystemInitializeRequest`：
      - `username`, `password`, `confirmPassword`, `nickname`, `email`, `phone`, `avatar`。
    - `websiteForm: WebsiteInfo`：
      - `name`, `description`, `icon`, `logo`, `spaLoadingIcon`。
    - 提交狀態：
      - `isSubmittingAdmin`：Step1 提交時的 loading 狀態（目前僅 UI 遲延，無實際 API）。
      - `isSubmittingWebsite`：Step2 提交與初始化 API 呼叫期間的 loading 狀態。

- **前端驗證與互動細節**
  - Step1：
    - 使用 `yup` schema 驗證帳號密碼與是否勾選隱私協議。
    - 額外提供密碼強度檢查：
      - 依長度、數字、小寫字母、大寫字母等規則計分，映射到不同顏色與文案（weak / medium /
              strong）。
      - 供使用者即時了解密碼安全程度，但並不直接阻止提交（提交依照 schema 與
              `agreedToTerms`）。
  - Step2：
    - 確保網站名稱必填，其餘欄位為選填.
    - 透過 `BdUploader + apiUploadInitFile` 處理 Logo /
          Icon 檔案上傳，前端只保存回傳的檔案 URL（或標識）。

- **初始化 API 呼叫：`apiInitializeSystem`**
  - 由 `useInstall().submitWebsiteConfig()` 發起：
    - 在呼叫前會將 `adminForm.confirmPassword` 設為與 `password` 相同.
    - 組合 `initializeData`：
      - 管理員相關欄位：
        - `username`, `password`, `confirmPassword`.
        - 選填欄位（若為空字串則轉為 `undefined`）：`nickname`, `email`, `phone`, `avatar`.
      - 網站設定相關欄位：
        - `websiteName`, `websiteDescription`, `websiteLogo`, `websiteIcon` 對應自
                  `websiteForm`.
  - API 回應處理：
    - 若回傳 `result.isInitialized === true`：
      - 顯示成功 toast（標題 `install.successTitle`，描述
              `install.steps.complete.successMessage`）。
      - 若回傳 `result.token`：
        - 呼叫 `userStore.setToken(result.token)` 設定登入 token.
        - 接著 `await userStore.getUser()` 拿到目前使用者資訊.
      - 重新載入全域設定：
        - `await appStore.getConfig()`.
        - `await appStore.getSystemInfo()`.
      - 回傳 `true` 給呼叫端，進入 Step3 完成頁.
    - 若 `result.isInitialized` 不為 true 或沒有回傳：
      - 回傳 `false`，使畫面停留在 Step2 以便使用者修正.
    - 發生例外時：
      - 捕捉錯誤並顯示錯誤 toast（標題 `install.failureTitle`，內容為錯誤訊息或
              `install.failureDescription`）。
      - 回傳 `false`.
    - finally 區塊：
      - 重設 `isSubmittingWebsite` 為 `false`.
      - 若 `appStore.siteConfig?.webinfo?.version` 存在，會以 `fetch` 向
              `EXTENSION_API_URL/building-ai/{version}` 回報安裝統計（非阻塞性呼叫）。

### 3.1 後端資料流與寫入目標（摘要）

- **API 入口與 DTO**
  - 前端 `apiInitializeSystem`：
    - `POST /consoleapi/system/initialize` → 轉發到後端
          `@ConsoleController("system")`、`@Post("initialize")`.
  - 後端 `initializeDto`：
    - 基於 `RegisterDto`（使用者註冊 DTO）去掉 `terminal`，並新增：
      - `avatar?`, `websiteName?`, `websiteDescription?`, `websiteLogo?`, `websiteIcon?`
              等欄位，均附帶字串驗證.

- **SystemService.initialize 的核心步驟**
  - 建立超級管理員（root）帳號：
    - 使用 `userService.hashPassword` 加密密碼.
    - 透過 `userService.create` 或類似方法，在 `User` 實體上寫入：
      - `username`, `password`, `isRoot=YES`,
              `nickname`（預設「超级管理员」）、`email`、`phone`、`avatar`（使用 dto 或隨機預設）、`source=CONSOLE`、`userNo`
              等欄位.
  - 寫入網站資訊設定：
    - 使用 `dictService.set(key, value, { group: "webinfo" })` 設定：
      - `name`、`description`、`logo`、`icon`（分別對應前端的
              `websiteName`、`websiteDescription`、`websiteLogo`、`websiteIcon`）。
  - 標記系統已初始化：
    - `dictService.set("isInitialized", true, { group: SYSTEM_CONFIG })`.
    - 此旗標是 `getSystemInfo()` 與前端 `checkSystemInitialization()` 的判斷來源.
  - 自動登入與權限載入：
    - 透過 `rolePermissionService.getUserRoles(user.id)` 與 `getUserPermissions(user.id)`
          載入角色／權限資訊.
    - 使用 `authService.userTokenService.createToken(...)`
          產出登入 token 與過期時間，並記錄終端（PC）、IP 與 user-agent.
    - 更新管理員 `lastLoginAt` 欄位.
  - 回傳給前端的資料：
    - `isInitialized: true`.
    - `token`, `expiresAt`.
    - `user`：包含使用者基本資訊（移除 password）與 `role`、`permissions`.

- **錯誤處理**
  - `SystemService.initialize` 捕捉到內部錯誤時，會使用：
    - `HttpErrorFactory.badRequest(error.message, error.data)` 拋出.
  - 前端目前僅顯示 `error.message` 至 toast 中，`error.data`
      的結構可在未來有需要時再開新 todo 深入整理（例如 mapping 成更友善的錯誤提示）。

## 4. 與全域邏輯的關聯

> 目標：說明安裝流程如何與整個系統的「已初始化／未初始化」邏輯配合，以及與全域狀態（appStore /
> userStore）的關係.

- **路由層：`route.global.ts` 與 `/install`**
  - 在每次路由切換時，global middleware 都會先呼叫：
    - `const initRedirect = await appStore.checkSystemInitialization(to.path)`。
  - `checkSystemInitialization` 根據 `systemInfo.isInitialized` 與目標路徑決定：
    - 系統尚未初始化且當前不在 `/install` → 回傳 `"/install"` 進行導向。
    - 系統已初始化且當前在 `/install` → 回傳 `/`（或其它預設路徑），避免再次進入安裝流程。
    - 其它情況 → 回傳 `null` 代表可繼續後續 middleware 與路由邏輯。

- **安裝成功後的狀態更新**
  - 成功呼叫 `apiInitializeSystem` 後：
    - 若有回傳 token：`userStore` 將持有有效登入狀態，後續 console 或其他受保護頁面可直接使用。
    - `appStore.getConfig()` 與 `appStore.getSystemInfo()` 會更新：
      - 站點資訊（名稱、Logo、描述等），影響 `app.vue` 中的 Head / SEO 與全域樣式。
      - 系統資訊（是否已安裝等），影響 `checkSystemInitialization` 的判斷結果。

- **再次訪問 `/install` 的行為（預期）**
  - 在系統已初始化的情況下：
    - `checkSystemInitialization("/install")` 應回傳 `/`
          之類的路徑，引導使用者回首頁或正常入口，而不是重新跑安裝精靈。

## 5. 重要畫面與組件

> 目標：列出安裝流程中最關鍵的畫面／組件，方便之後針對 UI 或行為做修改時有明確入口。

- **主要頁面**
  - `app/pages/install/index.vue`：
    - 控制 `currentStep`（0–3），並承載 Step0–Step3 四個子元件。
    - 設定 layout / auth meta，控制整個安裝畫面的外觀與權限。

- **步驟元件**
  - `components/step-0.vue`：
    - 歡迎畫面，負責動態標題動畫與開始安裝的 call-to-action。
  - `components/step-1.vue`：
    - 管理員帳號建立表單，包含密碼強度顯示與隱私協議勾選。
  - `components/step-2.vue`：
    - 網站資訊與品牌設定（名稱、描述、Logo、Icon），並提供返回上一步與提交下一步的按鈕。
  - `components/step-3.vue`：
    - 安裝完成提示，提供前往首頁與控制台的快速導向。

- **輔助元件與工具**
  - `components/true-focus.vue`：
    - 用於 Step0 的標題動畫效果，提升安裝歡迎頁的互動感。
  - `BdUploader`、`BdInputPassword` 等共用元件：
    - 處理檔案上傳（avatar、logo、icon）與安全的密碼輸入體驗。

## 6. TODO 與未決問題

- [ ] 閱讀 `app/pages/install/**` 詳細程式碼，補齊：
  - 路由與畫面流（第 2 節）。
  - 狀態與資料流（第 3 節）。
  - 全域邏輯關聯與畫面清單（第 4、5 節）。
- [ ] 確認與後端 API（初始化／安裝相關）的對應關係，必要時在這裡補上簡要列表，詳細欄位仍交由後端／API 文件維護。
