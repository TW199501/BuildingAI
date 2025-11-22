# 3ELF_API_OVERVIEW - API 總覽（給 AI 協作者）

## 1. 預期產生哪些文件（樹狀）

在 `## 0. 一律先讀的規則與流程` 底下新增了小節：

`### 0-1. 預期會產生的文件清單（樹狀）`

內容是固定「建議文件集合」，用你喜歡的樹狀格式：

```text
docs/
├─ ELF_EXPRESS_RULES.md       ← 團隊通用規則（已存在）
├─ ELF_AI_ONBOARDING.md       ← 本檔，AI 上手流程（已存在）
├─ PROJECT_OVERVIEW.md        ← 專案總覽（由 AI 建立／維護）
├─ FRONTEND_ARCH.md           ← 前端／UI 架構（由 AI 建立／維護）
├─ ui-flow-<feature>.md       ← 針對「少數重點功能」的 UI / Data flow（必要時才新增）
└─ todo/
   └─ todoYYYY-MM-DD-XX.md    ← 每日／每主題 todo 紀錄檔
```

---

## 2. 限制「檔案爆炸」的規則

同一節下面我加了兩條說明，直接解你剛說的問題：

- **不要為同一個網站／專案大量新增不同架構檔**：

  - 已經有 `PROJECT_OVERVIEW.md`、`FRONTEND_ARCH.md` 時：
    - 優先「在原檔增修小節」，
    - 不要再開 `overview-2.md / arch-2.md / notes-*.md` 這種新頂層檔。
  - 細節、實驗、一次性檢查 → 一律寫在「當日 todo 檔」。

- **`ui-flow-<feature>.md` 只給少數真的需要深入理解的功能**：

  - 例如：datetime refactor、設計器、授權流程。
  - 一般小功能／單次 bugfix 只寫 todo，不開新 `ui-flow-*.md`。

旁邊還有一句總結：

> 頂層文件保持精簡且穩定（overview／arch／少數 ui-flow）。日常變更與實驗集中寫在 `docs/todo/` 下面，避免「檔案越生越多」變成累贅。

---

## 3. 之後怎麼對 AI 下指令

你可以這樣要求任何新 AI：

- 先讀 [docs/ELF_EXPRESS_RULES.md](cci:7://file:///d:/app2/ELF-JNPF-BackEnd/jnpf-web-vue3-5.0-tw/docs/ELF_EXPRESS_RULES.md:0:0-0:0) 和 [docs/ELF_AI_ONBOARDING.md](cci:7://file:///d:/app2/ELF-JNPF-BackEnd/jnpf-web-vue3-5.0-tw/docs/ELF_AI_ONBOARDING.md:0:0-0:0)。
- 之後**只能**在上面那個樹狀清單裡的檔案中工作：
  - 如果要新增文件，只能是：
    - `PROJECT_OVERVIEW.md`（若不存在）
    - `FRONTEND_ARCH.md`（若不存在）
    - 某個必要的 `ui-flow-<feature>.md`
    - 每日的 [todoYYYY-MM-DD-XX.md](cci:7://file:///d:/app2/ELF-JNPF-BackEnd/jnpf-web-vue3-5.0-tw/docs/todo/todoYYYY-MM-DD-XX.md:0:0-0:0)
  - 其他說明／實驗，一律寫進 todo，而不是再發明新頂層 md。

---

### 狀態總結

- [ELF_AI_ONBOARDING.md](cci:7://file:///d:/app2/ELF-JNPF-BackEnd/jnpf-web-vue3-5.0-tw/docs/ELF_AI_ONBOARDING.md:0:0-0:0)：已加入「預期文件樹狀清單」＋「禁止檔案爆炸」的明確說明。
- [todoYYYY-MM-DD-XX.md](cci:7://file:///d:/app2/ELF-JNPF-BackEnd/jnpf-web-vue3-5.0-tw/docs/todo/todoYYYY-MM-DD-XX.md:0:0-0:0)：已有完整欄位範本，對應 `ELF_EXPRESS_RULES` 的第 8 節。

之後只要 AI 有遵守這兩份檔案，就會「少量固定檔案＋大量集中在 todo」，不會再把同一個網站拆成一堆零散 md。

## API_OVERVIEW

> 說明：本檔是「單一」API 總覽與索引，對應 `ELF_AI_ONBOARDING.md` 中的設計。目標是告訴人類與 AI：API 大致如何分區、對應哪些前端模組，以及正式 API 文件在哪裡。 **不**在這裡列出每一支 endpoint 的所有欄位與細節，那些細節請以後端正式文件或每日 todo 為主。

## 1. API 文件來源與權威

- **後端正式 API 文件（若有）**
  - 若專案提供 Swagger / OpenAPI / Postman / `apiDoc.md` 等，請在此小節補上：
    - 連結 URL 或檔案路徑。
    - 版本或產出時間（若可得知）。
- **本檔的角色**
  - 只做「導覽索引」與「前端模組對照表」，不是完整規格書。
  - 若正式 API 文件與本檔敘述有衝突，以正式 API 文件為準。

## 2. API 模組與前端 views 對照（總覽）

> 詳細一對一對應可參考：`docs/src-module-map.md`。
>
> 這裡只列出主要分區與大致職責，方便快速定位相關程式碼。

### 2.1 類別與路徑（示意範本）

```text
src/
├─ api/                       ← 前端呼叫後端 API 的封裝
│  ├─ system/                 ← 系統管理相關 API（使用者、角色、組織、授權…）
│  ├─ systemData/             ← 系統資料、資料介面等 API
│  ├─ onlineDev/              ← 線上開發、表單/流程/列表設計等 API
│  ├─ extend/                 ← 擴展業務模組 API
│  └─ ...                     ← 其他業務或基礎設施 API
└─ views/                     ← 對應頁面模組
   ├─ system/                 ← 系統管理相關頁面
   ├─ systemData/             ← 系統資料、資料介面相關頁面
   ├─ onlineDev/              ← 線上開發相關頁面
   ├─ extend/                 ← 擴展業務相關頁面
   └─ ...
```

> 實際的子模組與檔案名稱，請以 `src-module-map.md` 与 `src/api/`、`src/views/` 目錄為準。

### 2.2 常見 API 模組說明（請依實際專案補充）

- **system 模組**
  - 典型職責：帳號管理、角色／權限、組織架構、登入／登出、選單與路由設定。
  - 常見對應 views：`src/views/system/**`。
- **systemData / dataInterface 模組**
  - 典型職責：系統資料維護、資料介面（DataInterface）、資料連接、外部 API 代理等。
  - 常見對應 views：`src/views/systemData/**`，例如 DataInterface 列表、欄位設定畫面。
- **onlineDev 模組**
  - 典型職責：低程式碼線上開發相關 API，如表單設計、流程設計、列表（ColumnDesign）、應用發佈等。
  - 常見對應 views：`src/views/onlineDev/**`。
- **extend 模組**
  - 典型職責：業務擴展模組（如 CRM、專案管理等，依實際專案而定）。
  - 常見對應 views：`src/views/extend/**`。

> 任何新業務模組，請在此區段以相同格式簡短補上一小項，維持單檔總覽。

## 3. 權限、認證與共用行為（簡要）

> 詳細流程可在 `docs/core-infra.md` 或其他基礎架構文件中說明，這裡只放 API 面向的重點。

- **認證／授權相關**

  - 一般會有一組「認證 API」（登入、刷新 token、取得使用者資訊）與「權限 API」（取得角色、權限樹、可見菜單等）。
  - 請在此簡要描述：
    - 認證大致流程（例如：登入 → 儲存 token → 帶 token 呼叫其他 API）。
    - 權限資訊如何從 API 取得並寫入前端 store（可引用 `docs/core-infra.md` 的段落）。

- **錯誤格式與例外處理**

  - 若後端有統一的錯誤回應格式（例如 `{code, message, data}`），請在此描述：
    - 正常與錯誤回應的大致格式。
    - 前端 interceptor 或 wrapper 大致怎麼處理錯誤（可以指向實作檔案，而不是貼 code）。

## 4. 如何延伸或修改本檔

- **新增模組時**
  - 請直接在「2.2 常見 API 模組說明」區段補一個小項：
    - 模組名稱與路徑（例如 `src/api/report/**` / `src/views/report/**`）。
    - 短短 2 ～ 3 行說明其職責即可。
- **有新的權限／認證機制變更時**
  - 在第 3 節中補上新機制的簡要說明，並在對應的 todo 檔中詳細記錄變更過程與風險評估。

> 原則：`API_OVERVIEW.md` 保持在「一眼看懂 API 如何分區、去哪裡看詳細文件」的層級。具體 endpoint 規格、payload 欄位、錯誤細節都留給正式 API 文件或各自的 todo 紀錄處理。

## 5. 當你（AI）處理 API 相關需求時

1. **先確認模組範圍**

   - 判斷需求或 bug 屬於哪一個 API 模組（`basic` / `system` / `permission` / `systemData` / `onlineDev` / `extend` / `workFlow` ...）。
   - 對應到 `src/api/<module>/` 與 `src/views/<module>/` 的檔案位置。

2. **閱讀必要文件與程式碼**

   - 文件：
     - `docs/ELF_EXPRESS_RULES.md`
     - `docs/ELF_AI_ONBOARDING.md`
     - 本檔 `docs/API_OVERVIEW.md`
     - 視需要再看 `docs/src-module-map.md`、`docs/core-infra.md`。
   - 程式碼：
     - 相關的 `src/api/<module>/**` 與 `src/views/<module>/**`。
     - 若涉及認證／錯誤處理，再看 `src/utils/http/**`、`src/utils/auth/**`、`src/store/modules/user.ts` 等。

3. **在 todo 檔中紀錄整個過程**

   - 依照 `ELF_EXPRESS_RULES.md` 第 8 節與 `docs/todo/todoYYYY-MM-DD-XX.md` 範本，寫：
     - [問題標題]、[背景與重現步驟]、[相關檔案路徑與定位]
     - [分析與方案]、[實作重點摘要]、[驗證與結果]

4. **只在必要時更新本檔**

   - 新增或調整 API 模組時，在「2.2 常見 API 模組說明」中補一小項即可。
   - 認證／權限／錯誤處理有重大變更時，更新第 3 節的概要描述。
   - 不在本檔記錄單次 bug 排查細節，那些寫在當日 todo 檔。
