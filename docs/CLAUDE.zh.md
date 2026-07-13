# LibreChat

## 專案概觀

LibreChat 是一個 monorepo，包含以下主要工作區：

| 工作區 | 語言 | 端 | 依賴 | 用途 |
|---|---|---|---|---|
| `/api` | JS（舊有） | 後端 | `packages/api`、`packages/data-schemas`、`packages/data-provider`、`@librechat/agents` | Express 伺服器 — 盡量減少此處的變更 |
| `/packages/api` | **TypeScript** | 後端 | `packages/data-schemas`、`packages/data-provider` | 新的後端程式碼放在這裡（僅限 TS，供 `/api` 使用） |
| `/packages/data-schemas` | TypeScript | 後端 | `packages/data-provider` | 資料庫模型／結構，可供後端各專案共用 |
| `/packages/data-provider` | TypeScript | 共用 | — | 前後端共用的 API 型別、端點、data-service |
| `/client` | TypeScript/React | 前端 | `packages/data-provider`、`packages/client` | 前端 SPA |
| `/packages/client` | TypeScript | 前端 | `packages/data-provider` | 前端共用工具程式 |

`@librechat/agents`（主要後端依賴，同一團隊維護）的原始碼位於 `/home/danny/agentus`。

---

## 工作區邊界

- **所有新的後端程式碼都必須以 TypeScript 撰寫**，並放在 `/packages/api`。
- 將 `/api` 的變更降到最低（僅作為薄薄一層 JS 包裝，呼叫 `/packages/api`）。
- 資料庫相關的共用邏輯放在 `/packages/data-schemas`。
- 前後端共用的 API 邏輯（端點、型別、data-service）放在 `/packages/data-provider`。
- 從專案根目錄建置 data-provider：`npm run build:data-provider`。

---

## 程式碼風格

### 命名與檔案組織

- 儘可能使用**單字檔名**（例如 `permissions.ts`、`capabilities.ts`、`service.ts`）。
- 當需要多個單字時，優先將相關模組分組到**單字目錄**下，而非使用多字檔名（例如 `admin/capabilities.ts` 而非 `adminCapabilities.ts`）。
- 目錄本身已提供上下文 — 使用 `app/service.ts` 而非 `app/appConfigService.ts`。

### 結構與清晰度

- **避免巢狀**：儘早回傳、扁平化程式碼、最小縮排。將複雜操作拆分為命名清楚的輔助函式。
- **函式優先**：純函式、不可變資料、優先使用 `map`／`filter`／`reduce` 而非命令式迴圈。只有在明顯改善領域建模或狀態封裝時才採用 OOP。
- 除非絕對必要，**不使用動態 import**。

### DRY（不要重複自己）

- 將重複邏輯抽取為工具函式。
- 為 UI 模式建立可重用的 hooks／高階元件。
- 使用參數化輔助函式，而非近似重複的函式。
- 重複出現的值使用常數；以設定物件取代重複的初始化程式碼。
- 共用驗證器、集中式錯誤處理、業務規則的單一真實來源。
- 以介面／型別擴充共用基底定義，建立共用型別系統。
- 為外部 API 互動建立抽象層。

### 迭代與效能

- **盡量減少迴圈** — 尤其是對訊息陣列等在整個程式碼庫中頻繁被迭代的共用資料結構。每多一次遍歷，在大規模情況下都會累加成本。
- 儘可能將連續的 O(n) 操作合併為單一遍歷；若工作可以合併，切勿對同一個集合迴圈兩次。
- 選擇能減少迭代需求的資料結構（例如以 `Map`／`Set` 取代 `Array.find`／`Array.includes` 做查找）。
- 避免不必要的物件建立；考量空間與時間的取捨。
- 防止記憶體洩漏：小心處理閉包、釋放資源／事件監聽器、避免循環參照。

### 後端資料庫效能

- 在請求啟動與首次頁面載入的路徑上，留意連續（序列化）的資料庫讀取。
  當資料庫與應用伺服器距離較遠時，多次往返 MongoDB 會顯著增加延遲。
- 優先透過輔助函式傳遞已載入的請求／使用者／設定資料，
  而不是重複讀取相同的使用者、角色、租戶或主體資料。
- 當兩個讀取彼此獨立時，並行啟動它們，並在回傳資料前，
  以授權或驗證結果作為回應的關卡。
- 在並行化讀取時，保持授權、權限與租戶檢查在語意上完全一致。
  預先讀取（speculative reads）必須維持在已驗證使用者或租戶的範圍內，
  且在驗證成功之前不得寫入回應。

### 型別安全

- **絕不使用 `any`**。所有參數、回傳值與變數都要有明確型別。
- **限制使用 `unknown`** — 避免使用 `unknown`、`Record<string, unknown>` 以及 `as unknown as T` 的型別斷言。`Record<string, unknown>` 幾乎總是代表缺少明確的型別定義。
- **不要重複定義型別** — 在建立新型別之前，先確認專案中（尤其是 `packages/data-provider`）是否已存在相同型別。應重用並擴充既有型別，而非建立重複定義。
- 適當使用聯合型別、泛型與介面。
- 所有 TypeScript 與 ESLint 的警告／錯誤都必須處理 — 不得留下未解決的診斷訊息。

### 註解與文件

- 撰寫自我說明的程式碼；不加入敘述程式碼行為的行內註解。
- 僅在邏輯複雜／不直觀，或需要提供公開 API 的 IntelliSense 提示時使用 JSDoc。
- 簡短說明使用單行 JSDoc，複雜情況使用多行。
- 除非絕對必要，避免使用獨立的 `//` 註解。

### Import 順序

Import 分為三個區塊：

1. **套件 import** — 依行長度由短到長排序（`react` 永遠排在最前面）。
2. **`import type` import** — 依行長度由長到短排序（先列套件型別，再列本地型別；子群組之間長度重新計算）。
3. **本地／專案 import** — 依行長度由長到短排序。

多行 import 以所有行的總字元數計算長度。合併來自同一模組的值 import。務必使用獨立的 `import type { ... }` — 切勿在值 import 中內嵌 `type`。

### JS/TS 迴圈偏好

- **盡量減少迴圈使用。** 優先採用單一遍歷的轉換方式，避免重複迭代同一份資料。
- 效能關鍵或依賴索引的操作使用 `for (let i = 0; ...)`。
- 簡單的陣列迭代使用 `for...of`。
- 僅在列舉物件屬性時使用 `for...in`。

---

## 前端規則（`client/src/**/*`）

### 在地化（Localization）

- 所有面向使用者的文字都必須使用 `useLocalize()`。
- 僅更新 `client/src/locales/en/translation.json` 中的英文鍵值（其他語言由外部自動化處理）。
- 語意化鍵值前綴：`com_ui_`、`com_assistants_` 等。

### 元件

- 所有 React 元件都使用 TypeScript，並正確 import 型別。
- 使用具語意的 HTML 與 ARIA 標籤（`role`、`aria-label`）以符合無障礙需求。
- 將相關元件分組至功能目錄中（例如 `SidePanel/Memories/`）。
- 使用 index 檔案以保持匯出簡潔。

### 資料管理

- 功能 hooks：`client/src/data-provider/[Feature]/queries.ts` → `[Feature]/index.ts` → `client/src/data-provider/index.ts`。
- 所有 API 互動皆使用 React Query（`@tanstack/react-query`）；在 mutation 後正確地讓查詢失效（invalidation）。
- QueryKeys 與 MutationKeys 定義於 `packages/data-provider/src/keys.ts`。

### Data-Provider 整合

- 端點：`packages/data-provider/src/api-endpoints.ts`
- Data service：`packages/data-provider/src/data-service.ts`
- 型別：`packages/data-provider/src/types/queries.ts`
- 動態 URL 參數使用 `encodeURIComponent`。

### 效能

- 在大規模情況下優先考量記憶體與速度效率。
- 大型資料集使用游標分頁（cursor pagination）。
- 正確設定相依陣列（dependency array）以避免不必要的重新渲染。
- 善用 React Query 的快取與背景重新擷取（background refetching）。

---

## 開發指令

| 指令 | 用途 |
|---|---|
| `npm run smart-reinstall` | 安裝相依套件（若 lockfile 有變更）並透過 Turborepo 建置 |
| `npm run reinstall` | 全新安裝 — 清除 `node_modules` 並重新安裝 |
| `npm run backend` | 啟動後端伺服器 |
| `npm run backend:dev` | 以檔案監控模式啟動後端（開發用） |
| `npm run build` | 透過 Turborepo 建置所有編譯後程式碼（並行、具快取） |
| `npm run frontend` | 依序建置所有編譯後程式碼（舊版備援方式） |
| `npm run frontend:dev` | 啟動具 HMR 的前端開發伺服器（3090 埠，需先啟動後端） |
| `npm run build:data-provider` | 在變更後重新建置 `packages/data-provider` |

- Node.js：v24.16.0
- 資料庫：MongoDB
- 後端執行於 `http://localhost:3080/`；前端開發伺服器於 `http://localhost:3090/`

---

## 測試

- 框架：**Jest**，依各工作區分別執行。
- 從各自的工作區目錄執行測試：`cd api && npx jest <pattern>`、`cd packages/api && npx jest <pattern>` 等。
- 前端測試：置於元件旁的 `__tests__` 目錄；使用 `test/layout-test-utils` 進行渲染。
- 涵蓋 UI／資料流程的載入中、成功與錯誤狀態。

### 測試理念

- **使用真實邏輯，而非模擬。** 以真實依賴執行實際的程式碼路徑。模擬（mocking）是最後手段。
- **使用 spy 而非 mock。** 驗證真實函式是否以預期的參數與次數被呼叫，而不取代底層邏輯。
- **MongoDB**：使用 `mongodb-memory-server` 建立真實的記憶體內 MongoDB 實例。測試實際的查詢與結構驗證，而非模擬的資料庫呼叫。
- **MCP**：對伺服器、傳輸層與工具定義使用真實的 `@modelcontextprotocol/sdk` 匯出項目。反映真實情境，不要 stub SDK 內部實作。
- 僅模擬你無法控制的部分：外部 HTTP API、有速率限制的服務、非確定性的系統呼叫。
- 大量使用 mock 是程式碼異味（code smell），而非測試策略。

---

## 格式化

盡可能使用自動修正功能，修正所有格式化相關的 lint 錯誤（結尾空白、Tab、換行、縮排）。所有 TypeScript／ESLint 的警告與錯誤都**必須**解決。
