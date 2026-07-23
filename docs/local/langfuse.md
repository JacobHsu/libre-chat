# Langfuse 訊息回饋追蹤（按讚/倒讚）

> 官方 / 參考來源：
> - [langfuse.com](https://langfuse.com/)（Langfuse 官網）
> - [cloud.langfuse.com](https://cloud.langfuse.com)（Cloud 版註冊 / Project 設定）
> - [Langfuse Scores 文件](https://langfuse.com/docs/scores/overview)

Langfuse 是第三方 LLM 可觀測性（observability）平台，記錄每次對話的 trace（prompt、回應、token 用量、延遲），並可附加「評分」（score）。LibreChat 本身**不**內建按讚紀錄的統計後台——使用者在聊天室按 👍/👎 時，資料一定會寫進 MongoDB 的 `messages.feedback` 欄位，但要看聚合統計、依 trace 回溯完整對話、依模型/時間篩選，需要另外接 Langfuse。本文件說明如何在本機 `docker compose` 環境啟用 Langfuse Cloud，並驗證按讚資料有確實送過去。

## 1. 資料流程

```
使用者在訊息按 👍/👎
  → PUT /api/messages/:conversationId/:messageId/feedback
      → 寫入 MongoDB messages.feedback { rating, tag?, text? }
      → sendFeedbackScore()：POST Langfuse /api/public/scores
          name: "user-feedback"，value: 1（thumbsUp）或 0（thumbsDown）
          traceId 對應到該訊息原本產生回應時的 Langfuse trace
```

- 寫 Mongo一定會發生；送 Langfuse 是 best-effort（失敗只記 log，不影響按讚本身）。
- Assistants endpoint 的訊息沒有確定性 trace，會跳過送分數（[messages.js:444-463](../../api/server/routes/messages.js#L444-L463)）。
- 相關程式：[feedback.ts](../../packages/api/src/langfuse/feedback.ts)（送分數）、[destinations.ts](../../packages/api/src/langfuse/destinations.ts)（決定要送去哪個 Langfuse 專案）、[message.ts:81-99](../../packages/data-schemas/src/schema/message.ts#L81-L99)（Mongo schema）。

## 2. 建立 Langfuse Cloud 專案與金鑰

1. 到 [cloud.langfuse.com](https://cloud.langfuse.com) 註冊帳號（免費方案額度通常足夠評估用途）。
2. 建立一個 Project（例如 `librechat-dev`）。
3. Project Settings → **API Keys** → Create new API key，會拿到一組：
   - Public Key（`pk-lf-...`）
   - Secret Key（`sk-lf-...`）
4. 記下你註冊時選的資料區域（EU / US / JP），下一步要對應到 `LANGFUSE_BASE_URL`：

| 區域 | `LANGFUSE_BASE_URL` |
|---|---|
| EU（預設） | `https://cloud.langfuse.com` |
| US | `https://us.cloud.langfuse.com` |
| JP | `https://jp.cloud.langfuse.com` |

## 3. 填入 .env

本 repo 根目錄的 [.env](../../.env) 已經有對應欄位（原本註解掉），位置在 **`# Langfuse Tracing` 區塊**（約第 132-142 行）：

```env
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_BASE_URL=
```

把上一步拿到的值填進去即可，例如：

```env
LANGFUSE_PUBLIC_KEY=pk-lf-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
LANGFUSE_SECRET_KEY=sk-lf-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
LANGFUSE_BASE_URL=https://cloud.langfuse.com
```

> `.env` 是本機私密設定，本來就在 `.gitignore` 裡，不會被提交。同樣的三個欄位在 [.env.example](../../.env.example) 裡保留為註解範本（供其他人參考格式），不要把真實金鑰寫進 `.env.example`。

只需要這三個變數，**不需要**動下面的 `LANGFUSE_FANOUT_*` 區塊——那組是給多租戶（multi-tenant）架構用的 OTel fanout gateway，本機單機部署用不到，詳見 [otel/langfuse-fanout/README.md](../../otel/langfuse-fanout/README.md)。

## 4. 套用設定

[docker-compose.yml:30-33](../../docker-compose.yml#L30-L33) 把 `.env` 直接 bind mount 進 `LibreChat` 容器的 `/app/.env`，所以**不需要重新 build**，重啟 `api` 服務讓它重新讀檔即可：

```powershell
docker compose restart api
```

## 5. 驗證

**(a) 確認 trace 有送出去**

正常對話一輪後，到 Langfuse 專案的 **Tracing → Traces**，應該會看到新的 trace 陸續出現（trace 名稱/metadata 會帶 conversationId、endpoint 等）。如果完全没有 trace，代表金鑰或 base URL 設定有誤，先處理這個再看 feedback，見下方疑難排解。

**(b) 觸發一次按讚**

1. 在 LibreChat 前端隨便一則 AI 回覆下按 👍 或 👎。
2. **正確的查看路徑**：Langfuse 左側導覽 **Scores**（網址 `https://cloud.langfuse.com/project/<你的 projectId>/scores`，projectId 可從瀏覽器網址列目前所在頁面複製），應該會看到一筆 `Trace Name: AgentRun`、`Level: Trace`、`Name` 以 `user-feedback` 開頭、`Value: True`（thumbsUp）或 `False`（thumbsDown）。

   > ⚠️ **常見誤區**：不要從 Traces 列表點進某則 trace 後，在右側 peek 面板裡切到 **Scores** 分頁查看——那個分頁只顯示綁定在你點開的那個 observation（span）上的分數。LibreChat 送分數時只帶 `traceId`、不帶 `observationId`（見 [messages.js:446-448](../../api/server/routes/messages.js#L446-L448)），分數是掛在**整個 trace 層級**，只有走上面說的「左側導覽 Scores 總覽頁」或「trace 最頂層（不點進任何 observation）的 Scores 分頁」才看得到，點進子 span 的 Scores 分頁會誤以為「沒送出去」。
3. 也可以在 **Scores** 總覽頁直接依 `user-feedback` 篩選，看所有訊息的按讚/倒讚分佈與趨勢圖，這就是「評估按讚紀錄」實際會用到的畫面。

**(c) 交叉比對 MongoDB（可選）**

```powershell
docker exec -it chat-mongodb mongosh LibreChat --eval "db.messages.find({ 'feedback.rating': { \$exists: true } }, { messageId: 1, 'feedback': 1 }).limit(5)"
```

確認 Mongo 裡的 rating 跟 Langfuse Scores 頁面看到的一致。

## 6. 疑難排解

- **按了讚，Langfuse 完全没反应**：先確認是不是查看路徑錯誤（見上方 5(b) 的常見誤區），這是實際最常踩到的坑。若確認路徑沒錯還是沒有，才往下查：
  - `docker logs LibreChat --tail 50 | Select-String langfuse` 只能看到**失敗**時的 `logger.error` 訊息——本 repo `.env` 預設 `DEBUG_CONSOLE=false`，成功時的 `logger.debug`（例如 `[langfuse] central feedback score sent for trace ...`）不會出現在 `docker logs`，而是寫進本機 [logs/debug-YYYY-MM-DD.log](../../logs)：
    ```powershell
    Select-String langfuse logs\debug-$(Get-Date -Format yyyy-MM-dd).log
    ```
    看到 `feedback score sent for trace <traceId>` 就代表已經成功送出，回 Langfuse 用該 `traceId` 直接查（見 5(b)）。
  - 若連 debug log 裡都完全没有 `[langfuse]` 這行，代表 `sendFeedbackScore` 沒被觸發或 Promise 還沒跑完，常見原因是 `LANGFUSE_BASE_URL` 跟金鑰註冊的區域不一致（例如金鑰是 US 專案，但 `LANGFUSE_BASE_URL` 沒改成 `https://us.cloud.langfuse.com`），Langfuse 會回 401/404 並記成 `logger.error`。
- **有 trace 但沒有 Scores**：確認按讚的訊息不是走 Assistants endpoint（這類訊息目前設計上不送 feedback score，見上方第 1 節）。
- **完全沒有 trace（連對話本身都没送）**：代表 `LANGFUSE_PUBLIC_KEY`/`LANGFUSE_SECRET_KEY` 沒被讀到，或者 `LANGFUSE_TRACING_ENABLED=false` / `LANGFUSE_SAMPLE_RATE=0` 這類取樣設定被關掉了（本 repo 預設沒有設這兩個變數，即預設開啟）。先 `docker exec LibreChat sh -c "printenv | grep LANGFUSE"` 確認容器內確實讀到金鑰。
- **改了 .env 沒生效**：確認有執行第 4 節的 `docker compose restart api`，且改的是 repo 根目錄的 `.env`（不是 `.env.example`）。

## 參考資料

- [feedback.ts](../../packages/api/src/langfuse/feedback.ts) — 送出 `user-feedback` score 的實作
- [destinations.ts](../../packages/api/src/langfuse/destinations.ts) — 決定送去哪個 Langfuse 專案（central / tenant）的邏輯
- [config.ts](../../packages/api/src/langfuse/config.ts) — trace 本身的 Langfuse Run 設定
- [messages.js:426-475](../../api/server/routes/messages.js#L426-L475) — 按讚 API endpoint
- [message.ts:81-99](../../packages/data-schemas/src/schema/message.ts#L81-L99) — MongoDB `feedback` 欄位 schema
- [Langfuse Scores 文件](https://langfuse.com/docs/scores/overview)
