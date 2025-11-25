# UI Flow - Console：AI Provider 管理（草稿）

> 狀態：已完成第一輪 `console/ai/provider` 相關頁面閱讀，以下內容僅針對「AI
> Provider 管理」這個子系統的前端 UI
> flow 與資料流做整理；後端 provider 模組與 secret 模組的實作細節暫不展開。

## 1. 流程目的與範圍

- 目的：
    - 提供系統管理者在 console 中管理「AI 模型供應商（Provider）」的介面，決定：
        - 可以使用哪些第三方 LLM Provider（OpenAI、DeepSeek 等）。
        - 各 Provider 綁定哪一組 Secret／API Key。
        - 每個 Provider 支援哪些 model 類型，以及是否啟用顯示在系統中。
- 範圍：
    - 主要頁面：
        - `packages/web/buildingai-ui/app/pages/console/ai/provider/list.vue`。
    - 主要對話框／modal：
        - `packages/web/buildingai-ui/app/pages/console/ai/provider/edit.vue`。
    - 關聯但不深入的其他頁面：
        - Secret / API Key 管理頁（`useRoutePath("secret:list")` 導向處）。
        - 模型管理與 Agent 管理頁，因會消費這些 Provider 設定，但不在本 flow 中展開。

## 2. 路由與畫面流

> 目標：描述使用者進入 console 後，如何進到 AI
> Provider 管理頁、查看列表、進行新增／編輯／刪除與啟用／停用等操作的畫面流程。

- **入口路由與頁面**
    - 路由（推測）：`/console/ai/provider` 對應 `console/ai/provider/list.vue`。
    - 使用者通常從 console 主選單（AI / Provider 類別）進入此頁。

- **列表頁版面結構（`list.vue`）**
    - 最外層容器：`.provider-list-container pb-5`。
    - 上方 **搜尋與操作列**（sticky 在頂部）：
        - 元件：
            - `UInput` 綁 `searchForm.keyword`：
                - placeholder 為 i18n 文案 `ai-provider.backend.search.placeholder`。
                - `@change="getLists"`：關鍵字變更後重新拉取 provider 列表。
            - `USelect` 綁 `searchIsActive`（實際寫入 `searchForm.isActive`）：
                - items：
                    - all（全部）。
                    - show（顯示中）。
                    - hide（隱藏／停用）。
                - `@change="handleIsActiveChange"`：內部會轉換成
                  `searchForm.isActive = undefined | true | false`，再呼叫 `getLists()` 刷新。
            - 右側操作區（`md:ml-auto` 對齊右邊）：
                - 全選 checkbox + 已選數量顯示：
                    - `UCheckbox` 模型值依 `isAllSelected` / `isIndeterminate` 計算。
                    - 文案顯示 `selectedProviders.size / providers.length`。
                - 批次刪除按鈕（包在 `AccessControl` 裡）：
                    - `UButton`（color=error, variant=subtle），點擊呼叫
                      `handleBatchDelete`，會彈出確認 modal，成功後重新載入列表。
                - 新增 Provider 按鈕（同樣有 AccessControl）：
                    - `UButton`（primary）點擊 `handleAddProvider`，打開 `edit.vue` modal。
    - 中間 **卡片列表區**：
        - 當 `!loading && providers.length > 0`：
            - `BdScrollArea`（高度 `h-[calc(100vh-12rem)]`）包一個 `grid`：
                - `ProviderCard` v-for 渲染每一個 Provider：
                    - props：`provider`、`selected` 狀態。
                    - emits：`@select` / `@delete` / `@edit` / `@view-models` / `@toggle-active`
                      對應列表頁的方法。
        - 當 `loading`：顯示 loading spinner 與載入中文文案。
        - 當 `providers.length === 0` 且非 loading：
            - 顯示空狀態圖示、文案與一個「新增第一個 Provider」的 UButton（呼叫
              `handleAddProvider`）。
    - 底部 **統計列（sticky 在底部）**：
        - 僅在 `providers.length > 0` 時顯示。
        - 顯示：
            - 已選個數 `selectedProviders.size`。
            - Provider 總數（透過 i18n 文案表示）。

- **新增／編輯 Provider 的畫面流**
    - 從列表頁點擊：
        - 新增：
            - 按「新增 Provider」按鈕 → 呼叫 `handleAddProvider()` → `mountProviderEditModal()`。
        - 編輯：
            - 在 ProviderCard 中點擊「編輯」 → 呼叫 `handleEditProvider(provider)` →
              `mountProviderEditModal(provider.id)`。
    - `mountProviderEditModal` 內部：
        - 使用 overlay：`const modal = overlay.create(ProviderEdit)`。
        - 開啟 modal：`const instance = modal.open({ id: providerId })`。
        - 等待 modal 關閉：`const shouldRefresh = await instance.result`，若 `true` 則重新呼叫
          `getLists()` 重載列表。
    - 使用者在 `edit.vue` 中填寫或修改表單後：
        - 點擊「建立」或「更新」按鈕送出，成功會 `emits("close", true)`，讓列表頁知道需要
          `getLists()` 刷新。

- **刪除與批次刪除流程**
    - 卡片中的「刪除」按鈕：
        - 呼叫 `handleDeleteProvider(provider)` → 轉成單一 id 呼叫 `handleDelete(id)`。
    - 批次刪除：
        - 勾選多個 Provider（透過 `selectedProviders` Set 管理），再按「批次刪除」。
        - `handleBatchDelete()` 將 `selectedProviders` 轉成陣列，丟給 `handleDelete(selectedIds)`。
    - `handleDelete` 會：
        - 先呼叫 `useModal({...})` 顯示確認框（紅色 error 樣式）。
        - 若確認：
            - 單一 id → `apiDeleteAiProvider(id)`。
            - 多筆 id → `apiBatchDeleteAiProviders(idList)`。
        - 成功後：
            - 顯示 toast 成功訊息。
            - 清空 `selectedProviders`。
            - 再呼叫 `getLists()` 重新載入列表。

- **啟用／停用 Provider 的流程**
    - 在 ProviderCard 中會觸發 `@toggle-active="handleToggleProviderActive"`。
    - `handleToggleProviderActive(providerId, isActive)`：
        - 呼叫 `apiToggleAiProviderActive(providerId, isActive)`。
        - 成功後顯示「啟用成功 / 停用成功」的 i18n 訊息，並重新 `getLists()`。

## 3. 狀態與資料流

> 目標：整理 Provider 管理列表頁與編輯 modal 的主要 state 物件、webapi 呼叫與它們之間的關係。

### 3.1 列表頁（`list.vue`）的 state 與 webapi

- **搜尋條件與篩選**
    - `searchForm: AiProviderQueryParams`（shallowReactive）：
        - `keyword: string`：搜尋關鍵字。
        - `isActive?: boolean`：是否啟用。
    - `searchIsActive`（shallowRef）：用來對應 UI Select：
        - 值：`"all" | true | false`。
    - `handleIsActiveChange()`：
        - 若選擇 all → `searchForm.isActive = undefined`。
        - 若選擇 show / hide → 設為 true / false。
        - 然後呼叫 `getLists()` 重新載入列表。

- **Provider 列表資料**
    - `providers: shallowRef<AiProviderInfo[]>([])`。
    - `getLists` 透過 `useLockFn` 包裝：
        - `apiGetAiProviderList(searchForm)` → 將回傳結果指派給 `providers.value`。
        - `loading` 狀態由 `useLockFn` 提供。

- **選取狀態與批次操作**
    - `selectedProviders: ref<Set<string>>(new Set())`：記錄目前已勾選的 Provider id。
    - 單筆卡片勾選事件：`handleProviderSelect(provider, selected)`：
        - `selected === true` 時：`selectedProviders.add(id)`。
        - `selected === false` 時：`selectedProviders.delete(id)`。
    - 全選／取消全選：`handleSelectAll(value)`：
        - 若 value 為 true：對 `providers` 每筆呼叫 add。
        - 否則清空 Set。
    - `isAllSelected` / `isIndeterminate` 為 computed：
        - 幫助決定 UCheckbox 的顯示狀態（全選／半選／未選）。

- **刪除相關資料流**
    - `handleDelete(id: string | string[])`：
        - 先彈出確認 modal。
        - 成功確認後：
            - 若是陣列 → `apiBatchDeleteAiProviders(ids)`。
            - 若是單一 id → `apiDeleteAiProvider(id)`。
        - 然後：
            - `selectedProviders.clear()`。
            - `getLists()` 重新載入。

- **新增／編輯 modal 的開啟與關閉**
    - overlay：
        - `const modal = overlay.create(ProviderEdit)`。
        - `modal.open({ id: providerId })` 會回傳一個有 `result` 的 instance promise。
    - 當 `edit.vue` 中成功送出時會 emit `close(true)`，使得列表頁收到 `shouldRefresh = true`，接著
      `getLists()`。

### 3.2 編輯 modal（`edit.vue`）的 state 與 webapi

- **props 與路由整合**
    - props：`id?: string | null`，代表當前欲編輯的 Provider id。
    - 也會從 URL query 讀取 id：`route.query.id`。
    - `providerId = computed(() => props.id || (route.query.id as string) || null)`：
        - 允許 modal 以兩種方式取得目標 id（直接 props 或透過路由）。

- **表單 state**
    - `formData: shallowReactive<CreateAiProviderRequest>`：
        - `provider`：Provider 名稱／識別符（例如 `openai`）。
        - `name`：顯示名稱。
        - `bindSecretId`：綁定的 Secret／API Key 池 id。
        - `websiteUrl`：官方網站 URL。
        - `iconUrl`：圖示 URL（由 `BdUploader` 上傳並回填）。
        - `description`：說明文字。
        - `isActive: boolean`：是否啟用。
        - `sortOrder: number`：排序值。
        - `supportedModelTypes: string[]`：支援的 model 類別清單。
    - `detail: ref<AiProviderInfo | null>`：當編輯既有 Provider 時，存放從 API 取得的詳細資料。

- **表單驗證**
    - `providerSchema`（yup object）：
        - `provider`：必填。
        - `name`：必填。
        - `bindSecretId`：必填。
        - `supportedModelTypes`：至少要選 1 種 model type。
    - `UForm` 綁 `:schema="providerSchema"`，submit 會先做 yup 驗證。

- **支援模型類型資料**
    - `allModelTypes: shallowRef<ModelType[]>`：
        - 透過 `useAsyncData("model-types", apiGetAiProviderModelTypes)` 取得（在 script
          setup 頂層 await）。
        - 用於下拉多選：`UInputMenu`（`v-model="formData.supportedModelTypes"`）。

- **讀取 Provider 詳細資料**
    - `fetchDetail`（由 `useLockFn` 包裝）：
        - 若 `providerId.value` 存在，呼叫 `apiGetAiProviderDetail(providerId)`。
        - 把回傳結果塞進 `detail.value`，同時：
            - `modelTypes.value = data.supportedModelTypes`（僅用於展示）。
            - `useFormData(formData, data)`：將現有值覆蓋到 `formData`。
    - `onMounted(async () => providerId.value && (await fetchDetail()))`：
        - 當 modal 是「編輯模式」時，在掛載後讀取現有設定。

- **啟用狀態的 Radio group**
    - `isActiveString` 為 computed getter/setter：
        - get：`String(formData.isActive)`，對應 `'true' | 'false'`。
        - set：依 `'true' | 'false'` 更新 `formData.isActive`。
    - 在 template 中使用 `URadioGroup` 讓使用者以兩個按鈕切換啟用／停用。

- **跳轉到 API Key 管理**
    - `goToApiKeyManage()`：
        - `router.push({ path: useRoutePath("secret:list") })` 導向 Secret 管理頁。
        - 然後 `emits("close")` 關閉 modal。
    - 使用情境：
        - 當使用者尚未建立好 API Key 時，可從 Provider 編輯 modal 直接跳過去設定，再回來。

- **提交表單**
    - `submitForm`（包在 `useLockFn`）：
        - 先透過 UForm 的驗證（使用者 submit 時觸發）。
        - 若 `providerId.value` 存在：
            - 呼叫 `apiUpdateAiProvider(providerId.value, formData)` → 更新既有 Provider。
        - 若 `providerId.value` 為空：
            - 呼叫 `apiCreateAiProvider(formData)` → 建立新 Provider。
        - 成功後：
            - `toast.success(t("console-common.messages.success"))` 顯示成功訊息。
            - `emits("close", true)` → 通知上層（列表頁）應刷新列表。

## 4. 與全域邏輯與其他模組的關聯

> 目標：說明 Provider 管理 flow 如何與權限控制、Secret 管理、模型／Agent 管理與 chat
> C-API 組合成一條整體鏈路。

- **權限控制（AccessControl）**
    - 列表頁對「批次刪除」、「新增 Provider」這類操作，均包在 `AccessControl` 元件中：
        - 例如 `codes=["ai-provider.backends:delete"]`、`["ai-provider.backends:create"]`。
    - 若使用者沒有相應權限，按鈕可能被隱藏或 disabled（視 AccessControl 實作）。

- **與 Secret / API Key 管理的關係**
    - Provider 編輯頁的 API 配置區會要求選擇 `bindSecretId`：
        - 實際是從 Secret 池中選擇某個 KeyPool。
        - 若尚未建立 key pool，可藉由「前往 API Key 管理」按鈕跳到 console 的 secret 管理頁。
    - 這些 Secret 之後會被後端用來呼叫真實的 LLM Provider（在 chat C-API 深入文件已解釋）。

- **與模型與 Agent / Chat 流程的關係**
    - Provider 上設定的：
        - `supportedModelTypes`、`bindSecretId`、`isActive` 等資訊，會影響：
            - 能否在模型管理（AiModel）中掛載該 Provider。
            - chat C-API 在 `ModelValidationCommandHandler` 取 model + provider +
              secret 時是否能成功。
            - Agent 的設定與計費方式（某些 dashboard / logs 會顯示 Provider 對話統計）。

## 5. 重要畫面與關鍵元件

- **主要頁面**
    - `console/ai/provider/list.vue`：
        - 是 AI Provider 管理的主入口畫面，負責：
            - 搜尋／篩選 provider。
            - 顯示 provider 卡片列表與空／載入狀態。
            - 支援全選／批次刪除／啟用／停用操作。
            - 打開 Provider 編輯 modal 或 Model 列表 modal。

- **關鍵元件／對話框**
    - `console/ai/provider/components/provider-card.vue`：
        - 呈現單一 Provider 的卡片樣式，內含：
            - Provider 名稱、icon、啟用狀態等視覺資訊。
            - 勾選 checkbox、編輯按鈕、刪除按鈕、檢視模型按鈕（掛其他 modal）。
    - `console/ai/provider/edit.vue`：
        - 透過 `BdModal` 呈現 Provider 詳細設定表單。
        - 整合 UForm + yup 驗證、KeyPoolSelect、自訂上傳元件等。
    - 其他關聯元件（僅標註，不展開）：
        - `KeyPoolSelect`：選擇 API Key 池的共用元件。
        - `ModelModal`：檢視與 Provider 關聯的模型列表 modal。

## 6. TODO 與未決問題

- [ ] 若未來要深入 Provider module 的後端 API flow（Provider 與 Secret /
      AiModel 的關係），可另開 D-API 類型 TODO（類似 C-API 深入）。
- [ ] 視需求補充：ProviderCard / ModelModal / KeyPoolSelect 的細節，以支援更細緻的 console
      UX 設計或重構。
- [ ] 若要優化權限與操作提示（例如刪除前顯示「有幾個模型正在使用此 Provider」），可能需要再連動後端對依賴關係的查詢 API。
