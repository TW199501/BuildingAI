# 1-5ELF_AI_SCAN_EXAMPLE_BUILDINGAI - BuildingAI 0–50 掃描紀錄範例

> 本檔為示範用掃描紀錄，說明 AI 在 **0–50 分（探索期）**
> 時，針對 BuildingAI 專案應該如何紀錄「已讀內容、初步印象、未讀區塊與下一步計畫」。實際專案請依當時日期與實際閱讀範圍填寫。

---

### 掃描紀錄 2025-11-23 第 1 輪（約 0–50 分）

- 專案 / Repo 名稱：
    - BuildingAI

- 本輪掃描時間：
    - 2025-11-23 約 20 分鐘

- 專案目錄樹（含主要檔案；[x] 代表本輪已閱讀、[ ] 代表尚未閱讀）：

    . 📂 BuildingAI ├── 📄 LICENSE [x] ├── 📄 PRIVACY_NOTICE.md [ ] ├── 📄 README.md [x] ├── 📄
    README.zh-CN.md [x] ├── 📂 assets/ [ ] ├── 📂 docker/ [x] ├── 📂 docs/ [x] │ ├── 📄
    FRONTEND_ARCH.md [x] │ └── 📄 ... ├── 📂 extensions/ [ ] │ └── 📂 buildingai-simple-blog/ [ ]
    ├── 📂 packages/ [x] │ ├── 📂 api/ [ ] │ └── 📂 web/ │ └── 📂 buildingai-ui/ [ ] ├── 📂 public/
    [ ] ├── 📂 scripts/ [ ] ├── 📂 storage/ [ ] └── 📂 templates/ [ ]

- 初步印象（事實 / 推測 有區分）：
    - 專案大致用途：
        - （事實）`README.*` 說明這是一個包含後端 API 與前端 Console 的 monorepo，與 AI
          / 低程式碼相關。
        - （推測）可能是一個「建構 AI 應用或服務的平台」，前端提供 Console 後台、後端提供 AI
          / 帳務 / 租戶等服務。

    - 技術棧初步判斷：
        - （事實）README 與徽章顯示使用：
            - 後端：NestJS、TypeORM、PostgreSQL、Redis、BullMQ、TypeScript、Turbo。
            - 前端：Vue.js、NuxtJS、Vite、Nuxt UI。
        - （事實）`docker-compose.yml` 顯示有三個服務：
            - `redis`、`postgres`、`nodejs`，`nodejs` 中會執行 `pnpm start`。
        - （推測）前端與後端都透過 monorepo 由 `nodejs`
          容器啟動，開發模式是「volume 掛載原始碼、容器內跑 pnpm 指令」。

    - 專案結構印象：
        - （事實）`packages/api` 應為 NestJS API 服務所在。
        - （事實）`packages/web/buildingai-ui` 應為 Nuxt 前端專案。
        - （事實）`docs/FRONTEND_ARCH.md` 專門描述前端架構（Nuxt 啟動流程、路由等）。
        - （推測）未來還有 `docs/API_OVERVIEW`、`docs/…` 之類文件，用來逐步補齊 API 與架構說明。

- 尚未閱讀的重要區塊（樹狀＋勾選；閱讀完後可逐項改為 [x]）：

    目錄（尚未深入閱讀）├─ packages/ │ ├─ api/**[ ] （NestJS 模組、路由、資料庫實作）│ └─
    web/buildingai-ui/** [ ] （Nuxt 頁面、佈局、Pinia 狀態管理）├─ docs/ │ ├─
    onboarding-summary-from-early-todos.md [ ] │ └─ AI_COLLAB_RULES.md [ ] ├─
    docs/todo/2025-11-22-\*.md [ ] （了解之前的 Build / Docker 驗證歷程）├─ docker/ [ ]
    （其他 Docker / shell 腳本，若存在）└─ ...

- 下一輪閱讀計畫（進入 50–70 前想完成的事）：
    - 優先閱讀檔案 / 區塊（用樹狀列出要深入閱讀的節點）：

        下一輪優先閱讀├─ packages/api/ │ ├─ src/main.ts [ ] │ ├─ src/app.module.ts [ ] │ └─
        src/modules/**[ ] （只看有哪些模組）├─ packages/web/buildingai-ui/ │ ├─ nuxt.config.\* [ ] │
        ├─ app/app.vue [ ] │ └─ pages/** [ ] （整體分區）├─ docs/ │ └─
        onboarding-summary-from-early-todos.md [ ] （了解 Day1–2 核心重點）└─ ...

    - 希望在下一輪回答的問題：
        - API 模組是如何分領域（例如 AI / 帳務 / 用戶 / 租戶）？
        - Nuxt 前端的路由大致如何切頁籤（Console、設定、AI 相關頁面）？
        - Build / Docker 的實際啟動流程：是 dev-mode 還是 production build？

---

> 備註：
>
> - 本檔是「示範答案」，說明 0–50 階段應有的誠實度與細節。
> - 未來 AI 在新專案中，可以依 `1-4ELF_AI_UNDERSTANDING_TEMPLATE`
>   的掃描模板產出類似紀錄，但內容必須反映當前專案的真實狀態。
