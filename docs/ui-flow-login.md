# UI Flow - Login / Auth 登入流程（草稿）

> 狀態：已完成第一輪前端登入流程閱讀，以下內容根據 `app/pages/login/**`、user
> store 與全域 middleware 整理；與後端登入 API 欄位仍待未來有需要時再補充。

## 1. 流程目的與範圍

- 說明登入／驗證流程的目的：
    - 例：為受保護頁面（console、profile 等）提供安全的身份驗證與導向機制。
- 範圍（預期）：
    - 前端：`packages/web/buildingai-ui/app/pages/login/**`。
    - 狀態管理：`packages/web/@buildingai/stores/src/user.ts`（user store）。
    - 路由與中介層：`app/middleware/route.global.ts` 中與登入／權限／redirect 相關的邏輯。
    - 後端 API：（僅在需要時補充）登入、登出、取得當前使用者資訊等 endpoint。

## 2. 路由與畫面流

> 目標：從「未登入 → 進入登入頁 → 登入成功 → redirect 到目標頁」的完整畫面流。

- **入口路由與觸發情境**
    - 使用者直接訪問 `/login`：
        - `login/index.vue` 透過 `definePageMeta({ layout: "full-screen", auth: false })`：
            - 使用全螢幕登入佈局，且不需登入即可進入。
    - 使用者訪問需要驗證的路徑時：
        - 在 `route.global.ts` 的 `handleAuth()` 中：
            - 若 `!userStore.isLogin && to.meta.auth !== false && to.path !== ROUTES.LOGIN`：
                - 設定頁面 layout 為 `full-screen`。
                - 導向：`${ROUTES.LOGIN}?redirect=${to.fullPath}`。

- **登入頁整體結構（`login/index.vue`）**
    - 上方顯示站點 Logo 與名稱：
        - 若 `appStore.siteConfig.webinfo.logo` 存在，使用 `<NuxtImg>` 顯示自訂 logo 與名稱。
        - 否則顯示預設 `LogoFull` SVG。
    - 主要內容區：
        - 標題／副標：`login.title`、`login.subtitle`。
        - 一組「登入方式」按鈕列：
            - 根據 `appStore.loginSettings.allowedLoginMethods` 與 `LOGIN_COMPONENTS` 計算
              `loginMethods`：
                - 帳號登入（Account）。
                - 微信登入（WeChat）。
                - 其他方式（目前 Phone 被註解）。
            - 點擊按鈕會透過 `switchLoginMethod` 切換 `currentLoginMethod`，並顯示對應登入元件。
        - 下方顯示對應登入元件 `<component :is="currentComponent" ... />`：
            - 支援事件：
                - `@success="loginHandle"`：登入成功回調。
                - `@switch-to="switchLoginMethod"`：在元件內切換登入方式。
                - `@update:show-login-methods="showLoginMethods = $event"`：控制是否顯示上方登入方式按鈕列。
        - 頁面底部顯示版權資訊 `FooterCopyright`。

- **成功登入後的導向行為**
    - `loginHandle(data: LoginResponse & { hasBind: boolean })`：
        - 若系統要求強制綁定手機（`appStore.loginWay.coerceMobile * 1` 為真），且
          `!data.hasBind && !data.mobile`：
            - 呼叫 `userStore.tempLogin(data.token)` 暫存 token。
            - 將 `currentLoginMethod` 設為 `LOGIN_STATUS.BIND`，顯示綁定手機流程（`LoginBind`）。
        - 否則：
            - 直接呼叫 `userStore.login(data.token)`，觸發真正登入與 redirect 邏輯（詳見第 3 節）。

- **登入完成後再次訪問 `/login` 的行為**
    - 在 `route.global.ts` 的 `handleAuth()` 中：
        - 若 `userStore.isLogin && to.path === ROUTES.LOGIN`：
            - 若 `from.path !== ROUTES.LOGIN` → 回原路 `from.fullPath`。
            - 否則 → 回首頁 `ROUTES.HOME`。

## 3. 狀態與資料流

> 目標：整理登入流程中 user store 的狀態變化與與後端交換的主要資料。

- **user store 主要狀態（`@buildingai/stores/src/user.ts`）**
    - `token`：登入 token，來源為 cookie 或登入成功後設定。
    - `temporaryToken`：暫存 token，用於先登入但尚未完成手機綁定的情境。
    - `userInfo`：目前登入使用者資訊（由 `apiGetCurrentUserInfo()` 取得）。
    - `loginTimeStamp`：登入時間戳，用於判斷何時需要刷新 token。
    - `isAgreed`：是否勾選隱私／政策協議（在 WeChat 登入與隱私條款元件中使用）。
    - 計算屬性 `isLogin`：
        - 判斷 `token` 是否為非 null/undefined，用來辨識登入狀態。

- **login(newToken) 流程**
    - 呼叫來源：
        - `login/index.vue` 中的 `loginHandle`（帳號登入／微信登入／綁定手機完成之後）。
    - 步驟：
        1. 使用 `useRoute()` 取得當前路由與其 query。
        2. 若 `newToken` 為空 → 顯示錯誤訊息 `"Login error, please try again"` 並中斷。
        3. 呼叫 `setToken(newToken)`：
            - 更新 `token`。
            - 寫入 `STORAGE_KEYS.USER_TOKEN` cookie（有效期約 31 天）。
            - 同時更新登入時間戳。
        4. 更新登入時間戳 `setLoginTimeStamp()`，將目前時間寫入 cookie。
        5. `await nextTick()` 後呼叫 `getUser()` 取得使用者資訊並寫入 `userInfo`。
        6. 根據 `route.query.redirect` 決定導向：
            - 若 `redirect` 不存在，或等於 HOME/LOGIN → 透過
              `reloadNuxtApp({ ttl: 100, path: ROUTES.HOME })` 重新載入首頁。
            - 否則解析 `redirect`（支援帶 query 的 URL）為 `redirectPath`：
                - 若當前 `route.path === redirectPath` →
                  `reloadNuxtApp({ path: redirectUrl })`，確保頁面完整 reload。
                - 否則使用 `navigateTo(redirectUrl, { replace: true })` 導向目標路徑。

- **tempLogin(newToken) 流程**
    - 用於強制綁定手機的情境：
        - 在 WeChat 或其他登入方式回傳尚未綁定手機時，`loginHandle` 呼叫 `tempLogin`
          暫存 token，等待綁定完成後再正式登入。
    - 實作：僅將 `token.value = newToken`，不進行 cookie 寫入與 redirect。

- **登出（logout）流程**
    - 可選 `expire` 參數標示是否為 token 過期登出：
        - 若 `expire` 為真且尚未顯示過提示，會透過
          `useMessage().warning("Login expired, please login again")` 提示使用者，並在關閉提示時重置
          `onExpireNotice`。
    - 流程：
        - 呼叫 `clearToken()`：清除 token、temporaryToken 與登入時間戳，並刪除相關 cookie。
        - 若當前路由 `meta.auth !== false`（即需要驗證）：
            - 導向 `ROUTES.HOME`，並在 query 帶上 `redirect=<原路徑>` 方便使用者重新登入後回到原頁.

- **刷新 token（refreshToken）流程**
    - 在 `handleAuth()` 中，當 `userStore.isLogin` 且非 `/login` 時會呼叫
      `userStore.refreshToken()`：
        - 僅在 `token` 存在且 `loginTimeStamp` 非 0 時生效.
        - 若距離上次登入時間超過 7 小時，會：
            - 重新將目前 token 寫入 cookie、更新到期時間.
            - 呼叫 `setLoginTimeStamp()` 重設登入時間戳.

### 3.1 後端資料流與 API 摘要（帳號登入／微信登入／手機綁定／使用者資訊）

- **帳號登入 `/auth/login`**
    - 前端 `apiAuthLogin(params)`：
        - 透過 `useWebPost("/auth/login", params)` 呼叫 Web API.
    - 後端 `AuthWebController.login`：
        - `@Public() @Post("login")`，接收 `LoginDto`（`username`、`password`、`terminal` 等）。
        - 根據 `loginDto.terminal` 或預設 `UserTerminal.PC` 作為登入終端.
        - 呼叫 `authService.login(username, password, terminal, ip, userAgent)`.
    - 在 `AuthService` 中（結構與 `register` 類似）：
        - 驗證使用者是否存在與密碼是否正確，錯誤時使用 `HttpErrorFactory` 拋出對應錯誤碼與訊息.
        - 取得使用者角色／權限資訊，並透過 `userTokenService.createToken(...)`
          建立 token 與過期時間.
        - 更新 `lastLoginAt` 後回傳 `{ token, expiresAt, user: {..., role, permissions} }`
          給前端，對應 `LoginResponse` 型別.

- **取得目前使用者資訊 `/user/info`**
    - 前端 `apiGetCurrentUserInfo()`：
        - 透過 `useWebGet("/user/info", {}, { requireAuth: true })` 取得目前登入者資訊.
    - 後端 `UserWebController.getUserInfo`：
        - `@Get("info")`，接收 `@Playground() user`（由 auth middleware 解出 token 對應的使用者）。
        - 使用
          `userService.findOneById(user.id, { excludeFields: ["password"], relations: ["role"] })`
          讀取使用者與角色資訊.
        - 使用 `rolePermissionService.getUserPermissions(user.id)` 取得權限碼清單.
        - 回傳 `{ ...userInfo, permissions: hasPermissions }`，其中 `hasPermissions`
          為 0 或 1，代表是否擁有任一權限或為 root.

- **微信掃碼登入：`/auth/wechat-qrcode` 與 `/auth/wechat-qrcode-status/:scene_str`**
    - 前端：
        - `apiGetWxCode()` → `GET /auth/wechat-qrcode`，取得二維碼資訊 `WechatLoginCode`（`key`,
          `qrcode`, `url`）。
        - `apiCheckTicket({ key })` → `GET /auth/wechat-qrcode-status/:scene_str`，輪詢登入狀態
          `WechatLoginTicket`：
            - `is_scan: boolean`：是否已掃碼並完成登入.
            - `user: LoginResponse`：登入成功時回傳的使用者與 token 資料.
    - 後端 `AuthWebController`：
        - `@Get("wechat-qrcode")`：呼叫 `wechatOaService.getQrCode(expire_seconds)`
          產生帶場景值的臨時二維碼.
        - `@Get("wechat-qrcode-status/:scene_str")`：呼叫
          `wechatOaService.getQrCodeStatus(scene_str)` 回傳當前場景值的登入狀態與對應使用者.
        - 另有 `wechat-callback`
          系列 endpoint 處理微信伺服器的推送，更新 Redis 中的二維碼狀態，最終由 `getQrCodeStatus`
          告知前端是否登入成功.

- **SMS 與手機綁定：`/sms/sendCode`、`/user/bindMobile`**
    - 前端：
        - `apiSmsSend({ scene, mobile })` → `POST /sms/sendCode`，用於發送手機驗證碼（例如
          `scene: BIND_MOBILE`）。
        - `apiUserMobileBind({ type, mobile, code })` → `POST /user/bindMobile`，在 `login-bind.vue`
          中使用，成功後再透過 `loginHandle` 進行正式登入.
    - 後端：
        - 路徑已由前端 service 確認，但具體 controller/service 實作尚未在本 todo 中深入；未來若需對手機登入安全性或錯誤碼做調整，可再開新 todo 針對這兩個 endpoint 深挖.

## 4. 與全域路由邏輯的關聯

> 目標：說明登入狀態如何影響 `route.global.ts` 中的導向行為與 layout 設定.

- **未登入時的導向**
    - 在 `route.global.ts` → `handleAuth()`：
        - 若 `!userStore.isLogin && to.meta.auth !== false && to.path !== ROUTES.LOGIN`：
            - 將頁面 layout 設為 `full-screen`.
            - 回傳 `"/login?redirect=" + to.fullPath`，交由 Nuxt 導向登入頁並帶上 redirect 資訊.

- **已登入卻前往登入頁**
    - 同樣在 `handleAuth()` 中：
        - 若 `userStore.isLogin && to.path === ROUTES.LOGIN`：
            - 若 `from.path !== ROUTES.LOGIN` → 回上一頁 `from.fullPath`。
            - 否則 → 回首頁 `ROUTES.HOME`。

- **已登入訪問其他頁面**
    - 若 `userStore.isLogin` 且目標並非 `/login`：
        - 呼叫 `userStore.refreshToken()` 嘗試延長 token 有效期。

## 5. 重要畫面與組件

> 目標：列出登入相關頁面與核心表單元件，方便之後追程式碼或調整登入行為。

- **主要頁面**
    - `app/pages/login/index.vue`：
        - 控制登入方式（帳號、微信、綁定手機），並組合登入元件與登入結果處理邏輯。
        - 根據 `appStore.siteConfig` 顯示品牌 Logo 與系統名稱。

- **關鍵元件與子流程**
    - `components/account/account.vue`：
        - 負責「帳號登入子流程」，內部再切換三個子元件：
            - `login.vue`：帳號密碼登入表單（發出 `success` 時帶 `LoginResponse`）。
            - `register.vue`：帳號註冊流程。
            - `forget.vue`：忘記密碼／重置流程。
        - 透過 `@switch-component` 事件切換上述子元件，並用 `update:showLoginMethods`
          控制是否顯示上方登入方式按鈕列。
    - `components/wechat/wechat.vue`：
        - 負責「微信掃碼登入」：
            - `apiGetWxCode()` 取得二維碼 URL 與 key。
            - 使用 `usePollingTask` 週期性呼叫 `apiCheckTicket({ key })`
              檢查是否已在微信端完成登入。
            - 成功時 emit `success(LoginResponse)` 給父層，交由 `loginHandle` 處理。
            - 根據狀態顯示「登入成功、二維碼已過期、登入失敗」等提示，並提供刷新按鈕。
            - 若 `appStore.loginSettings.showPolicyAgreement` 為 true 且 `userStore.isAgreed`
              為 false，會以遮罩層阻止使用者直接掃碼，要求先勾選隱私協議（顯示 `PrivacyTerms`
              元件）。
    - `components/login-bind.vue`：
        - 在強制綁定手機情境下顯示：
            - 透過 `apiSmsSend` 發送驗證碼（`scene: SMS_TYPE.BIND_MOBILE`）。
            - 使用 `apiUserMobileBind` 綁定手機號碼。
            - 成功後 emit `success({ token: userStore.token, hasBind: true })`，讓父層再次透過
              `loginHandle` 進行正式登入與 redirect。

## 6. TODO 與未決問題

- [ ] 視需求補充：帳號登入與微信登入實際呼叫的後端登入 API 路徑與 payload／回應結構（目前僅在前端層面記錄 token 進入點與後續流程）。
- [ ] 若未來要支援更多登入方式（如 Phone / Google /
      Email 等），可在本檔新增對應子流程描述，並標記各方式共用的 user store 與 route
      middleware 行為。

- [ ] 閱讀 `app/pages/login/**` 以及 `user` store 的實作，補齊：
    - 路由與畫面流（第 2 節）。
    - 狀態與資料流（第 3 節）。
    - 與全域路由邏輯的關聯（第 4 節）。
    - 重要畫面與關鍵元件（第 5 節）。
- [ ] 視需要補充與後端登入／登出／取得使用者資訊 API 的對應關係。
