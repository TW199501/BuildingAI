# UI Flow - Chat 對話流程（草稿）

> 狀態：已完成第一輪 chat 前端流程閱讀，以下內容根據 `app/pages/chat/[id].vue`、`ask-assistant-chat` 元件、chat store 與相關 webapi 整理；與後端 chat API 實作細節暫不展開，僅在此記錄前端可見的資料流。

## 1. 流程目的與範圍

- 說明 chat 對話流程的目的：
  - 提供使用者與 AI 模型互動的主要介面，支援多會話、歷史訊息瀏覽、附件上傳、模型與 MCP 選擇等功能。
- 範圍：
  - 前端主頁面：`packages/web/buildingai-ui/app/pages/chat/[id].vue`。
  - chat UI 組成元件：
    - `components/ask-assistant-chat/chats-chats.vue`（會話列表）。
    - `components/ask-assistant-chat/chats-messages.vue`（訊息列表）。
    - `components/ask-assistant-chat/chats-prompt/chats-prompt.vue`（輸入列）。
    - `components/ask-assistant-chat/preview-sidebar/*`（預覽側邊欄）。
  - 狀態管理：
    - `app/stores/chat.ts`（pendingConversation 暫存）。
    - `useControlsStore()`（控制 sidebar 顯示與選中模型）。
  - 前端 webapi（僅列用途）：
    - `@buildingai/service/webapi/ai-conversation`：`apiGetAiConversation*`、`apiChatStream`、`apiGetChatConfig`、`apiGetQuickMenu` 等。

## 2. 路由與畫面流

> 目標：從「選擇一個會話 → 查看歷史訊息 → 與模型對話 →（可選）預覽引用內容」的完整畫面流。

- **入口路徑與頁面**
  - 路由：`/chat/:id` 對應 `app/pages/chat/[id].vue`。
  - `definePageMeta({ activePath: "/" })`：
    - 表示在主導航上仍視為首頁分支的一部分（方便高亮主選單）。

- **整體版面結構**
  - 最外層：
    - 左側固定寬度 sidebar：會話列表 `<ChatsChats />`。
    - 右側 `SplitterGroup`：
      - 主聊天區（訊息列表 + 輸入列）。
      - （可選）預覽側邊欄 `<PreviewSidebar />`，由 `usePreviewSidebar()` 控制開關與內容。

- **會話列表區（左側 `<ChatsChats />`）**
  - 載入：
    - `useAsyncData("chats", () => apiGetAiConversationList({ page: 1, pageSize: 50 }))`。
    - 將回傳的 `chats.value.items` 依 `updatedAt` 使用 `groupConversationsByDate` 分組顯示。
  - 當前選中會話：
    - 根據 `route.name === "chat-id"` 與 `route.params.id` 計算 `currentChatId`。
  - 手機／桌機行為：
    - 使用 `useMediaQuery` 判斷行動裝置，透過 `controlsStore.chatSidebarVisible` 控制 sidebar 顯示與折疊。
  - 會話操作：
    - 重新命名：打開對話框後呼叫 `apiUpdateAiConversation(id, { title })`，成功後 `refreshNuxtData("chats")`。
    - 刪除會話：呼叫 `apiDeleteAiConversation(id)` 後刷新列表（細節可在需要時再補）。

- **主聊天區（右側 SplitterPanel）**
  - 標題列：
    - 上方 `UInput` 顯示當前會話標題（`currentConversation?.title`）。
    - `@change` 事件會呼叫 `saveEditedName`，進而透過 `apiUpdateAiConversation` 保存新的標題。
  - 訊息滾動區：
    - 使用 `BdScrollArea` 包裹 `BdChatScroll`，後者負責 infinite scroll：
      - `:loading="loading"`、`:has-more="hasMore"`、`@load-more="loadMoreMessages"`。
    - `ChatsMessages` 負責渲染 `messages`（來自 `useChat`），同時支援：
      - 錯誤訊息顯示、工具輸出（MCP）、引用文件、推理展示等。
    - 滾動狀態：
      - 透過 `handleScroll` 與 `isAtBottom` 判斷是否在底部；
      - 當訊息更新或內容重繪（code highlight 等）時，若在底部則自動 `scrollToBottom()`。
  - 輸入區：
    - 底部 `<ChatsPrompt>` 作為輸入欄與附件區，詳細見第 3 節。
    - 下方再顯示一行 `chatConfig.welcomeInfo.footer` 文字作為版權／提示區。

## 3. 狀態與資料流

> 目標：整理 chat 流程中 messages/input/files 等狀態如何隨使用者操作與 API 呼叫變化。

- **初始載入與歷史訊息**
  - 在 `[id].vue` 中：
    - 以 `currentConversationId = route.params.id` 計算會話 ID。
    - 使用 `apiGetAiConversation(id, { page: 1, pageSize: 5 })` 取得第一批訊息：
      - `transform` 時會將 `data.items` 反轉，使時間順序為舊 → 新。
    - 若沒有資料，會直接 `throw createError({ statusCode: 404, ... })` 結束流程。
  - `hasMore`：
    - 初始為 `messagesData.total > messagesData.items.length`，代表是否還有更舊的訊息可載入。

- **使用 `useChat` 管理對話狀態**
  - 呼叫方式（簡化）：

    ```ts
    const { messages, input, files, handleSubmit, reload, stop, status, error } = useChat({
      id: currentConversationId.value,
      api: apiChatStream,
      initialMessages,
      body: {
        get modelId() { return selectedModel.value?.id },
        get mcpServers() { return JSON.parse(localStorage.getItem("mcpIds") || "[]") },
        saveConversation: true,
        get conversationId() { return currentConversationId.value },
      },
      onToolCall() { scrollToBottom() },
      onResponse(response) { if (response.status === 401) userStore.logout(true) },
      onError(err) { /* 顯示錯誤訊息 */ },
      onFinish() {
        userStore.getUser();
        refreshNuxtData("chats");
        refreshNuxtData(`chat-detail-${currentConversationId.value}`);
      },
    });
    ```

  - `messages`：
    - 初始值由歷史訊息 `messagesData.items` 轉換而來（將 `errorMessage` → `content`，並設定 `status`）。
    - 之後由 `useChat` 在 streaming 過程中追加或更新。
  - `input`：
    - 綁定 `<ChatsPrompt v-model="input" />`，使用者輸入文字即更新。
  - `files`：
    - 綁定 `<ChatsPrompt v-model:file-list="files" />`，由附件上傳邏輯更新（`usePromptFiles`）。
  - `status` / `error`：
    - 由 `useChat` 控制：
      - `status === "loading"` 代表訊息正在生成。
      - `error` 傳給 `ChatsMessages` 以顯示錯誤狀態。

- **載入更多歷史訊息**
  - `queryPaging`：`{ page: 1, pageSize: 5 }`。
  - `loadMoreMessages()`：
    - 若 `hasMore` 為 false 則不動作。
    - `queryPaging.page++` 後呼叫 `apiGetAiConversation(id, queryPaging)` 取得更舊訊息：
      - 將結果 `data.items.reverse()`，轉成 `AiMessage` 後 `messages.value.unshift(...newMessages)`。
    - 依 `data.total > messages.value.length` 更新 `hasMore`，錯誤時回退 page。

- **模型選擇與 MCP 選擇**
  - 模型：
    - 由 `controlsStore.selectedModel` 管理，透過 `<ModelSelect @change="controlsStore.setSelectedModel" />` 更新。
    - `useChat` 的 `body.modelId` 會讀取此選擇。
  - MCP servers：
    - `selectedMcpIdList` 維護目前選中的 MCP ID 陣列，同時透過 localStorage `mcpIds` 持久化。
    - `handleQuickMenu()` 會在 localStorage 與 state 之間同步 quick menu 對應的 MCP ID，並更新 `isQuickMenu` 狀態。
    - `useChat` 的 `body.mcpServers` 則會從 localStorage 讀取最終列表。

- **pendingConversation 暫存與自動發送**
  - chat store：`app/stores/chat.ts`：
    - `pendingConversation: { id, modelId, title, files? } | null`。
    - 提供 `setPendingConversation` / `clearPendingConversation` / `getPendingConversation`。
  - 在 `[id].vue` 的 `onMounted` 中：
    - 若目前 `messages.value.length === 0` 且 chatStore 中存在 `pendingConversation`、且 `pendingConversation.id === currentConversationId`：
      - 將 `pendingConversation.files` 指派到 `files.value`。
      - 將 `selectedModel.value.id` 設為 `pendingConversation.modelId`。
      - `await nextTick()` 後呼叫 `handleSubmitMessage(pendingConversation.title)`，自動送出預設問題。
      - 最後呼叫 `chatStore.clearPendingConversation()` 清空暫存。

## 4. 與全域邏輯與其他模組的關聯

> 目標：說明 chat 流程如何與 user store、controls store 與其他使用 `useChat` 的場景協作。

- **user store 與登入狀態**
  - 在 `useChat` 的 `onResponse` 中，若收到 401：
    - 會 `userStore.logout(true)`，並透過全域 route middleware 重新導向登入與首頁（邏輯詳見 `ui-flow-login.md`）。
  - 在 `onFinish` 時呼叫 `userStore.getUser()`：
    - 可能用於刷新使用者權限或餘額資訊，確保 UI 顯示最新狀態。

- **controls store：sidebar 與模型選擇**
  - `useControlsStore()` 控制：
    - `chatSidebarVisible`：在手機／桌機間切換時由 `chats-chats.vue` 的 `watch(isMobile)` 決定顯示或隱藏 sidebar。
    - `selectedModel`：透過 `<ModelSelect>` 設定，供 `useChat` 作為 `body.modelId`。

- **其他使用 `useChat` 的場景（預覽用，不屬於主 C-flow）**
  - `pages/console/ai/agent/components/preview-chat.vue`：
    - 為後台 Agent 設定提供「預覽聊天」：
      - `api: apiAgentChat`。
      - `body` 帶入 agent 的 modelConfig、datasetIds、rolePrompt 等設定。
      - `onUpdate` 會在首次 streaming 時取得 `conversation_id` 存入本地狀態。
  - `pages/public/agent/components/chat.vue`：
    - 公開分享的 Agent Chat 畫面：
      - 使用 `apiChatStream`，`body` 帶入 `publishToken`、`accessToken`、`billingMode` 與 formFields。
      - 會話列表與歷史訊息由 `apiGetMessages`、`apiGetAgentInfo` 等 webapi 處理。
  - 這些場景與主 chat 頁面共享相同的 `useChat` 行為模型，有助於之後重構或擴充 chat 能力。

## 5. 重要畫面與關鍵元件

> 目標：列出 chat 相關頁面與核心元件，方便日後調整行為或除錯時快速找到入口。

- **主要頁面**
  - `app/pages/chat/[id].vue`：
    - 單一會話的主畫面，負責：
      - 讀取初始歷史訊息、會話詳細資訊與 chatConfig。
      - 初始化 `useChat` 並將 messages/input/files 與 UI 元件串接。
      - 控制滾動行為、載入更多歷史訊息，以及與 chat store / controls store / user store 的互動。

- **關鍵元件**
  - `components/ask-assistant-chat/chats-chats.vue`：
    - 會話列表與搜尋／分組呈現，並處理重新命名與刪除行為。
  - `components/ask-assistant-chat/chats-messages.vue`：
    - 訊息氣泡、推理顯示、知識引用、MCP tool call、附件預覽等。
  - `components/ask-assistant-chat/chats-prompt/chats-prompt.vue`：
    - 輸入框、文案優化按鈕、附件上傳、快捷操作按鈕，以及 Enter/Shift+Enter 送出邏輯。
  - `components/ask-assistant-chat/preview-sidebar/*`：
    - 顯示被引用的文件、圖片或其他預覽內容，與 `usePreviewSidebar` 搭配使用。

- **輔助元件與工具**
  - `BdScrollArea`、`BdChatScroll`：
    - 提供具 infinite scroll 能力的滾動容器，與 `loadMoreMessages` 搭配。
  - `ModelSelect`、`McpToolSelect`：
    - 模型與 MCP 選擇器，影響 `useChat` 傳給後端的 `modelId`、MCP servers。

## 6. TODO 與未決問題

- [ ] 視需求補充：`useChat` composable 的內部實作細節（目前僅依使用介面推測行為）。
- [ ] 若未來要支援更多 chat 模式（例如多 modal 視窗、多 tab、並排比較等），可在本檔新增對應子流程描述，並標記與現有 `useChat`、chat store 的關聯。
- [ ] 若有需要深入分析 chat 相關後端 API，可另開 C-API 專用 todo，對 `/ai-chat/stream`、`/ai-conversations/**`、`/config/chat` 等端點做完整 call stack 梳理。
