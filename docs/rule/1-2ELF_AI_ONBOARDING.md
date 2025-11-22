# ELF EXPRESS - AI 專案上手流程（0 → 100）

本文件說明：當有新的 AI 加入 ELF EXPRESS 的任一專案時，應該如何從 0 開始理解專案，逐步做到「可以安全地修改程式碼」。

## 0. 一律先讀的規則與流程

1. **專案協作規則**

   - 檔案：`docs/ELF_EXPRESS_RULES.md`
   - 內容重點：
     - 最小改動、安全、可回滾。
     - 每日 todo markdown：`docs/todo/todoYYYY-MM-DD-XX.md`。
     - 分兩階段寫筆記：
       - 分析＋方案。
       - 修改內容＋驗證結果。
     - 技術棧一律依官方文件與標準 API。

2. **團隊開發流程圖（含 AI 協作）**

   - 範本檔：`docs/todo/todoYYYY-MM-DD-XX.md`
   - 使用 mermaid 描述：
     - 發現問題 → 讀規則／舊 todo → 分析＋風險 → 寫「分析與方案」→ 實作 → 驗證 → 在同一 todo 補「修改內容與驗證結果」。

> **任何新 AI 進專案，第一件事就是閱讀上述兩個檔案。**

### 0-1. 預期會產生的文件清單（樹狀）

對於同一個專案（或同一個網站），預設只需要產生**少量且固定種類**的文件。建議的文件結構如下：

```text
docs/
├─ ELF_EXPRESS_RULES.md       ← 團隊通用規則（已存在）
├─ ELF_AI_ONBOARDING.md       ← 本檔，AI 上手流程（已存在）
├─ PROJECT_OVERVIEW.md        ← 專案總覽（由 AI 建立／維護）
├─ FRONTEND_ARCH.md           ← 前端／UI 架構（由 AI 建立／維護）
├─ API_OVERVIEW.md            ← （可選）API 總覽／索引（單一檔案即可）
├─ ui-flow-<feature>.md       ← 針對「少數重點功能」的 UI / Data flow（必要時才新增）
└─ todo/
   └─ todoYYYY-MM-DD-XX.md    ← 每日／每主題 todo 紀錄檔
```

說明：

- **不要為同一個網站／專案大量新增不同架構檔**：
  - 若已有 `PROJECT_OVERVIEW.md`、`FRONTEND_ARCH.md`，請優先在這兩個檔案內增修小節，而不是再開新的「overview-2.md / arch-2.md / notes-\*.md」。
  - 對於細節或一次性的實驗，請優先記在**當日的 todo 檔**，而不是多開新的頂層文件。
- **API_OVERVIEW.md（可選）只作為 API 總覽／索引**：
  - 不列出每一支 endpoint 的所有欄位，只整理：
    - API 類別或領域 → 對應的前端模組（例如某資料夾或路由區段）。
    - 正式 API 文件放在哪裡（例如 Swagger 網址、`apiDoc.md` 等）。
  - 若專案已經有完整 API 文件（Swagger、`apiDoc.md`），請優先在 `PROJECT_OVERVIEW.md` 中說明「API 文件在哪裡」，必要時再新增一份輕量的 `API_OVERVIEW.md` 當導覽索引即可。
- **ui-flow-`<feature>`.md 僅用於少數「需要深入理解」的功能**：
  - 例如：datetime refactor、某個複雜設計器、核心授權流程。
  - 若只是小功能或一次性 bugfix，通常只需要 todo 紀錄，不需要獨立 `ui-flow-*.md`。

> 簡單講：
>
> - 頂層文件保持精簡且穩定（overview／arch／少數 ui-flow）。
> - 日常變更與實驗集中寫在 `docs/todo/` 下面，避免「檔案越生越多」變成累贅。

---

## 1. 建立「初始理解」的 todo 檔

第一次接手專案時：

1. 建立當日第一個 todo 檔，例如：

   - `docs/todo/todo2025-11-23-01.md`

2. 在該檔的「分析與方案」區段，AI 應先寫下：

   - 專案來源與目的（如果已知）。
   - 預計閱讀的檔案與目錄（掃描計畫）。
   - 預計產出哪些總覽文件（例如 `PROJECT_OVERVIEW.md`、`FRONTEND_ARCH.md`、`ui-flow-xxx.md` 等）。

3. 尚未修改任何程式碼，只是 **記錄「準備怎麼理解專案」**。

---

## 2. 產出專案總覽：PROJECT_OVERVIEW

**目標：** 讓任何人／AI 打開一份檔案，就能快速理解專案整體輪廓。

1. 建議檔名：

   - `docs/PROJECT_OVERVIEW.md`

2. AI 應閱讀的檔案（可依專案調整）：

   - `README.md`
   - 核心架構文件（若已有）：例如 `docs/*architecture*.md`、`docs/*overview*.md`
   - 主要建置／環境檔：`package.json`、`pnpm-lock.yaml`、`vite.config.*`、`tsconfig.*` 等。

3. 在 `PROJECT_OVERVIEW.md` 中至少包含：

   - 專案目標與 Domain（例如：低程式碼平台、ERP、物流系統…）。
   - 技術棧（前端、後端、DB、CI/CD 等）。
   - 目錄結構大綱：
     - `src/` 下各資料夾的職責說明。
     - 重要的 `docs/` 與 `config/` 檔位置。
   - 啟動與 build：
     - 開發指令（例如 `npm run dev`）。
     - build 指令（例如 `npm run build`）。
     - 任何特別的前置步驟（如 env 設定）。

4. 寫完後，在當日的 `todoYYYY-MM-DD-01.md` 中「修改內容與驗證」區段補充：

   - 讀了哪些檔、產出了哪些檔（例如：新增 `PROJECT_OVERVIEW.md`）。
   - 對專案的第一版理解摘要（幾行文字即可）。

---

## 3. 產出前端／UI 架構文件：FRONTEND_ARCH

**目標：** 清楚說明前端應用是如何被啟動、如何組成。

1. 建議檔名：

   - `docs/FRONTEND_ARCH.md`

2. AI 應額外閱讀的檔案：

   - 入口：`src/main.ts` 或 `src/main.js`
   - 路由：`src/router/**`
   - 狀態：`src/store/**`、`src/pinia/**` 等
   - 全域組件註冊：例如 `src/components/registerGlobComp.ts`
   - UI 架構相關 docs：如 `docs/ui-architecture.md`

3. 在 `FRONTEND_ARCH.md` 中至少包含：

   - App 啟動流程（main 檔流程圖或條列）。
   - Router 結構與主要模組（系統管理、業務模組、開發工具…）。
   - 狀態管理策略（Pinia/Vuex 模組、分層方式）。
   - 全域 UI 元件或 Design System（例如 Jnpf 元件群組、Ant Design Vue 的使用方式）。

4. 同樣在當日 todo 檔中更新「修改內容與驗證」區段：

   - 紀錄新增 `FRONTEND_ARCH.md`，並簡述其內容與用途。

---

## 4. 為特定功能寫 UI / Data Flow 文件

當有特定子系統或功能需要深入理解（例如本次 JNPF 的 datetime refactor）：

1. 建議檔名格式：

   - `docs/ui-flow-<feature>.md`
   - 例如：`docs/ui-flow-datetime-refactor.md`

2. AI 應：

   - 閱讀相關模組路徑（例如：`src/views/systemData/dataInterface/**`、`src/components/ColumnDesign/**`）。
   - 找出與該功能相關的關鍵檔案（元件、utils、hook 等）。

3. 在 `ui-flow-<feature>.md` 中：

   - 使用 mermaid 畫出：
     - 使用者操作流程（從哪個畫面進入、點哪些按鈕）。
     - 資料流向（API → store → 元件 → UI）。
   - 條列說明每個節點的大致職責。

4. 在對應主題的 todo 檔（例如 `todo2025-11-22-01.md`）的「分析與方案」或「修改內容與驗證」區段引用此文件：

   - `參考：docs/ui-flow-datetime-refactor.md`。

---

## 5. 每日 todo 的使用方式

> 一個主題 = 一個 todo 檔。同一檔有兩輪主要寫作：先寫「分析＋方案」，後寫「修改內容＋驗證」。

1. 檔名格式：

   - `docs/todo/todoYYYY-MM-DD-XX.md`
   - 如：`todo2025-11-22-01.md`、`todo2025-11-22-02.md`。

2. 第一輪：**分析與方案**

   - 說明要處理的問題／需求。
   - 目前觀察到的行為／錯誤。
   - 擬定的方案 A/B（含優缺點）。
   - 計畫閱讀／修改的檔案列表。

3. 第二輪：**修改內容與驗證**

   - 實際修改了哪些檔案、哪些段落（可簡要列出）。
   - 執行了哪些指令（`npm run dev`、`npm run build`、測試指令）。
   - 結果如何（成功／失敗，若失敗要貼關鍵錯誤訊息）。
   - 下一步的 TODO（若仍有未解問題）。

4. 若同一天有不同主題：

   - 為每個主題使用新的流水號：`-02`, `-03`…。
   - 例如：
     - `todo2025-11-22-01.md`：datetime refactor。
     - `todo2025-11-22-03.md`：v-code-diff / build 問題。

---

## 6. 給未來 AI 的標準啟動指令範例

當你在這個 repo 啟動一個新的 AI 協作時，可以用類似這樣的提示：

> 1. 請先完整閱讀 `docs/ELF_EXPRESS_RULES.md`。
> 2. 然後閱讀 `README.md`、`package.json`、`src/main.ts` 與任何現有的架構文件。
> 3. 建立 `docs/todo/todoYYYY-MM-DD-01.md`：
>    - 在「分析與方案」區段，寫下你對專案的初步理解與接下來的掃描計畫。
> 4. 依照上面的流程，協助我產出 `docs/PROJECT_OVERVIEW.md` 與 `docs/FRONTEND_ARCH.md`。
> 5. 之後每個新主題（例如某個 refactor 或 bugfix），請使用新的 `todoYYYY-MM-DD-XX.md` 按照「分析 → 實作 → 驗證」的方式紀錄整個過程。

只要你遵守這份 `ELF_AI_ONBOARDING.md` 加上 `ELF_EXPRESS_RULES.md`，就能從 0 穩定地把這個專案「讀懂」，再安全地進入實作階段。
