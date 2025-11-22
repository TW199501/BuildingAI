# API OVERVIEW（草稿）

> 狀態：草稿。僅根據目前已閱讀的模組與路由建立「鳥瞰圖級」總覽，用來當作後端 API 導覽地圖與後續深入文件的入口。詳細欄位與錯誤碼仍以各模組原始程式與專門文件為準。

## 1. 文件目的

- 說明 BuildingAI 後端 API 在「模組 / 路由前綴」層級的分佈與職責。
- 幫助開發者快速找到：
  - 某個前端 flow（例如 install / login / chat / console AI Provider）背後對應的後端模組。
  - 不同功能模組（auth / user / system / ai-chat / console api 等）的大致邊界與責任。
- 本文件**不維護**每個 API 的完整欄位與錯誤碼，只作為：
  - 「進入點索引」（go-to place）。
  - 連結到更細的 TODO / ui-flow / 深入文件。

## 2. 高階分層

- **Public / Web API**（面向前台 web / H5 / 小程式）：
  - 範例前綴：`/auth/**`、`/user/**`、`/ai-chat/**`、`/ai-conversations/**`。
- **Console API**（面向後台管理介面）：
  - 範例前綴：`/consoleapi/system/**`、AI Provider / Model / Secret 等管理功能。
- **Internal / Common 模組**：
  - 例如 auth、user、billing、dict 等，被 web 與 console 模組重用。

## 3. 已閱讀模組速記

> 這一節暫時只列出目前我們已在 TODO / ui-flow 文件中實際閱讀過的後端模組，之後可逐步補齊。

### 3.1 System 初始化（Install 流程）

- 模組位置：`packages/api/src/modules/system/**`
- 代表路由：
  - `POST /consoleapi/system/initialize`
- 職責摘要：
  - 在 install 流程中建立超級管理員帳號、寫入網站資訊（webinfo 字典）、標記系統已初始化並回傳登入 token。
- 關聯文件：
  - `docs/ui-flow-install.md`
  - `docs/todo/todo2025-11-22-03.md`

### 3.2 Auth / User 登入與使用者資訊

- 模組位置（摘要）：
  - `packages/api/src/common/modules/auth/**`
  - `packages/api/src/modules/user/**`
- 代表路由（webapi）：
  - `POST /auth/login`：帳號登入。
  - `GET /auth/wechat-qrcode`、`GET /auth/wechat-qrcode-status/:scene_str`：微信掃碼登入 ticket 流程。
  - `POST /sms/sendCode`、`POST /user/bindMobile`：手機綁定與簡訊驗證碼。
  - `GET /user/info`：取得當前登入使用者資訊。
- 職責摘要：
  - 統一處理登入、token 建立與驗證、第三方登入（微信）、手機綁定與使用者資訊查詢。
- 關聯文件：
  - `docs/ui-flow-login.md`
  - `docs/todo/todo2025-11-22-04.md`

### 3.3 AI Chat 與對話記錄（C-API）

- 模組位置：`packages/api/src/modules/ai/chat/**`
- 代表路由（webapi）：
  - `POST /ai-chat`：一般聊天請求。
  - `POST /ai-chat/stream`：SSE 流式聊天。
  - `POST /ai-chat/optimize-text`：文案優化。
  - `GET /ai-conversations`、`GET /ai-conversations/:id`、`GET /ai-conversations/:id/messages`：對話與訊息查詢。
  - 以及對話更新、刪除、置頂等相關路由。
- 職責摘要：
  - 串接 LLM Provider、MCP 伺服器與工具，處理聊天完成、多輪工具呼叫、扣點與對話／訊息的持久化。
- 關聯文件：
  - `docs/ui-flow-chat.md`
  - `docs/todo/todo2025-11-22-06.md`

### 3.4 Console：AI Provider 管理（前端對應的後端輪廓）

- 目前僅從前端角度知道：
  - 前端使用的 service 在 `@buildingai/service/consoleapi/**`（具體路徑與後端模組待後續補充）。
  - 涉及路由前綴類似 `consoleapi/ai-provider/**`（實際名稱待確認）。
- 代表行為：
  - 取得 Provider 列表、批次刪除、單筆刪除、啟用／停用 Provider、建立／編輯 Provider。
- 關聯文件：
  - `docs/ui-flow-console-ai-provider.md`
  - `docs/todo/todo2025-11-22-07.md`

## 4. TODO

- [ ] 擴充第 3 節，依模組（auth / user / system / ai-chat / billing / secret / ai-provider / ai-model / ai-agent 等）逐步補齊代表路由與職責摘要。
- [ ] 明確整理 console API 前綴（`/consoleapi/**`）與各子系統（Provider / Model / Secret / Agent / Billing...）的對應關係。
- [ ] 視需要從本檔連結到更多專門文件（例如未來可能的 `api-flow-*.md`）。
