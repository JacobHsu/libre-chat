# 開關 Endpoint 與「使用者自帶金鑰」提示

> 官方文件參考：
> - [librechat.ai/zh/docs/configuration/dotenv](https://www.librechat.ai/zh/docs/configuration/dotenv)（`ENDPOINTS` 環境變數；各 `*_API_KEY` 留空/`user_provided` 的行為）
> - [librechat.ai/zh/docs/configuration/pre_configured_ai/assistants](https://www.librechat.ai/zh/docs/configuration/pre_configured_ai/assistants)（`ASSISTANTS_API_KEY` / `user_provided`）
> - [librechat.ai/zh/docs/configuration/librechat_yaml/object_structure/custom_endpoint](https://www.librechat.ai/zh/docs/configuration/librechat_yaml/object_structure/custom_endpoint)（自訂 endpoint 的 `apiKey` 三種寫法：環境變數參照／`user_provided`／直接寫死金鑰）

模型選單裡「要不要出現某個 endpoint」和「該 endpoint 要不要跳出『設定 API 金鑰』提示」是**兩個互相獨立的設定**，很容易搞混。

## 1. `ENDPOINTS`：控制選單要顯示哪些 endpoint

在 [.env](../../.env#L246) 裡：

```
# ENDPOINTS=openAI,assistants,azureOpenAI,google,anthropic
```

這行預設是**註解掉**的（開頭有 `#`），代表 `ENDPOINTS` 這個環境變數沒有被設定。註解掉的內容只是官方範例，不會生效。

當 `ENDPOINTS` 未設定時，後端會 fallback 用寫死在程式碼裡的預設清單（[packages/data-provider/src/parsers.ts:77-86](../../packages/data-provider/src/parsers.ts#L77-L86)）：

```
openAI, agents, assistants, azureAssistants, azureOpenAI, google, anthropic, bedrock
```

要拿掉某個 endpoint（例如不想讓使用者看到 Assistants），必須**取消註解並列出自己要的清單**，例如：

```
ENDPOINTS=openAI,agents,google,anthropic
```

改完要重啟後端才會生效（見下方「套用設定」）。

## 2. `<ENDPOINT>_API_KEY=user_provided`：控制「設定 API 金鑰」提示

每個 endpoint 各自有一個對應的 API Key 環境變數（例如 `ASSISTANTS_API_KEY`、`OPENAI_API_KEY`、`ANTHROPIC_API_KEY`）。當它的值是特殊字串 `user_provided` 時，該 endpoint 會被標記為「需要使用者自己提供金鑰」，前端選單上就會出現「設定 API 金鑰」提示，並在 Settings → Provider Keys 顯示可輸入金鑰的欄位。

判斷邏輯是簡單的字串比對（[packages/api/src/utils/common.ts:36](../../packages/api/src/utils/common.ts#L36)）：

```ts
export const isUserProvided = (value?: string): boolean => value === AuthType.USER_PROVIDED;
```

本 repo 目前 [.env:454](../../.env#L454)：

```
ASSISTANTS_API_KEY=user_provided
```

若不想要求使用者自己輸入 Assistants 的金鑰，但仍想保留 Assistants 這個 endpoint，改成實際的 key（例如複用 `OPENAI_API_KEY` 的值）即可：

```
ASSISTANTS_API_KEY=sk-xxxx
```

同樣需要重啟後端才會生效（見下方「套用設定」）。

## 3. 套用設定：重啟

`.env` 只在後端啟動時被讀取一次，改完必須重啟才會生效。依你本機的啟動方式擇一：

**Docker Compose：**

```bash
docker compose down
docker compose up -d
```

只是改 `.env` 變數，不需要 `docker compose build`（沒改 image）。

**直接跑 `npm run backend`：**

停掉現有的 process（`Ctrl+C`），再重新執行：

```bash
npm run backend
```

若是用 `npm run backend:dev`（nodemon 監控檔案變更），修改 `.env` 本身**不會**觸發自動重啟，一樣要手動停掉重開。

### 驗證

重啟後開啟前端模型選單，確認：
- 不想要的 endpoint（如 Assistants）已經消失。
- 保留的 endpoint 旁的「設定 API 金鑰」提示是否符合預期。

若沒生效，優先檢查：
- `docker compose logs api`（或終端機輸出）有沒有 `.env` 解析錯誤。
- 是不是改到了非實際使用的 `.env`（跟 [.env.example](../../.env.example) 搞混）。

## 4. 兩者的差異

| 情境 | 要改的設定 |
|---|---|
| 完全不想讓使用者看到 Assistants 這個選項 | `ENDPOINTS`（拿掉 `assistants`） |
| 保留 Assistants，但不要跳出「設定 API 金鑰」 | 對應的 `_API_KEY`（改成真實金鑰，不要用 `user_provided`） |
| 保留 Assistants，且允許使用者自己輸入金鑰（多租戶／不想代管金鑰的情境） | 維持 `_API_KEY=user_provided` 不動 |

本 repo 目前採用第一種：[.env:246](../../.env#L246) 已設定 `ENDPOINTS=openAI,agents,google,anthropic`，Assistants 不會出現在選單中。

## 5. 實測驗證：金鑰「留空」跟「填真實金鑰」是兩種不同結果

官方文件（[librechat.ai/zh/docs/configuration/dotenv](https://www.librechat.ai/zh/docs/configuration/dotenv)）對每個 `*_API_KEY` / `GOOGLE_KEY` 的說明是：

> 留空以禁用此 endpoint（Google、OpenAI 等皆同）
> 設置為 `"user_provided"` 以允許使用者從 WebUI 提供自己的 API key

也就是「留空」跟「`user_provided`」是不同結果；本 repo 目前的 `.env` 剛好各踩到一種情境，實測如下：

### 5.1 `GOOGLE_KEY=#user_provided` → 整個 Google 選項消失

[.env:368](../../.env#L368)：

```
GOOGLE_KEY=#user_provided
```

這行**不是註解**——`#` 前面沒有空格，dotenv 在解析時仍會把 `=` 後面的 `#user_provided` 整段當成行內註解砍掉，實測解析結果是空字串：

```bash
docker exec LibreChat node -e "require('dotenv').config(); console.log('[' + process.env.GOOGLE_KEY + ']')"
# 輸出：[]
```

`GOOGLE_KEY` 變空字串後，[api/server/services/Config/loadAsyncEndpoints.js:34-53](../../api/server/services/Config/loadAsyncEndpoints.js#L34-L53) 的邏輯：

```js
const isGoogleKeyProvided = googleKey && googleKey.trim() !== '';  // false
// 沒有 GOOGLE_SERVICE_KEY_FILE，也沒有 api/data/auth.json，serviceKey 也是 null
const google = serviceKey || isGoogleKeyProvided ? { userProvide: googleUserProvides } : false;
// → google = false
```

`endpointsConfig.google` 整個變成 `false`，前端選單裡 **Google 這個選項直接消失**，不是只有「設定金鑰」提示不見——跟官方文件說的「留空以禁用此 endpoint」完全對應。

若原本想要的是「Google 保留、讓使用者自己輸入金鑰」，正確寫法是不帶 `#`：

```
GOOGLE_KEY=user_provided
```

### 5.2 `OPENAI_API_KEY=sk-proj-...`（真實金鑰）→ Endpoint 保留，只關掉「設定金鑰」

[.env:433](../../.env#L433) 目前是一組真實的 OpenAI key（非 `user_provided`）。這種情況下 [api/server/utils/handleText.js:127-133](../../api/server/utils/handleText.js#L127-L133) 的 `generateConfig()`：

```js
function generateConfig(key, baseURL, endpoint) {
  if (!key) {
    return false;                              // 留空 → 整個 endpoint 消失（同 5.1 的 Google 案例）
  }
  const config = { userProvide: isUserProvided(key) };  // 真實金鑰 → isUserProvided() 回傳 false
  ...
}
```

`key` 有值但不等於 `"user_provided"`，所以走的是 `{ userProvide: false, ... }` 這條路——**OpenAI 這個 endpoint 仍然留在選單裡**，只是 `userProvide: false` 讓前端不再顯示「設定 API 金鑰」的輸入框（因為系統認定伺服器端已經有金鑰了）。

### 5.3 三種狀態總結

| `.env` 寫法 | 解析後的值 | Endpoint 是否出現在選單 | 是否顯示「設定金鑰」 |
|---|---|---|---|
| 留空／整行註解掉 | 空字串 | ❌ 消失（`generateConfig`/`loadAsyncEndpoints` 回傳 `false`） | 不適用（選項都沒了） |
| `KEY=user_provided` | 字面字串 `user_provided` | ✅ 出現 | ✅ 顯示，使用者自行輸入 |
| `KEY=sk-xxxx`（真實金鑰） | 真實金鑰字串 | ✅ 出現 | ❌ 不顯示（視為伺服器已代管金鑰） |
| `KEY=#user_provided`（`#` 緊接在值前、無空格） | dotenv 仍會當作行內註解砍掉 → 空字串 | ❌ 消失（等同第一種） | 不適用 |

最後一列是本 repo 目前 `GOOGLE_KEY` 的真實案例——很容易誤以為「加 `#` 只是註解掉範例值，其他字元還在」，但 dotenv 的行內註解规則不要求 `#` 前面有空格，所以整段都被吃掉了，效果等同完全沒設定這個變數。
