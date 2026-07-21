# 本機 RAG（檔案搜尋 / rag_api + vectordb）

LibreChat 的「檔案搜尋」（File Search）功能讓你上傳文件後，用語意檢索（而非把整份文件塞進 context）回答跟文件內容相關的問題，也就是一般說的 RAG（Retrieval-Augmented Generation）。這個功能由兩個額外的容器提供：`rag_api`（負責切 chunk、embedding、檢索）和 `vectordb`（`pgvector`/Postgres，存向量）。本文件說明這兩個服務在本機 `docker compose` 環境下的架構、怎麼在對話裡正確觸發它、以及怎麼直接查詢向量資料庫內容做除錯。

## 架構

```
使用者上傳檔案
   → LibreChat api 容器 (RAG_API_URL=http://rag_api:${RAG_PORT:-8000})
       → rag_api 容器：切 chunk → 呼叫 embeddings API → 存進 vectordb
對話中提問
   → rag_api 對 vectordb 做語意查詢，撈出最相關的 chunk
   → 連同問題一起送給 LLM，回答時附上引用來源
```

[docker-compose.yml](../../docker-compose.yml) 已經把這兩個容器接好，本機 `docker compose up -d` 就會一起啟動，不需要額外設定：

- `api` 服務 `depends_on: rag_api`，並設定 `RAG_API_URL=http://rag_api:${RAG_PORT:-8000}`
- `vectordb` 用 `pgvector/pgvector` image，帳密在 `docker-compose.yml` 裡寫死（`myuser` / `mypassword` / db `mydatabase`）
- Embeddings 預設用 `OPENAI_API_KEY`（`.env` 裡設的那把），沒有另外設 `RAG_OPENAI_API_KEY` 的話會 fallback 用它

## 確認服務正常

```powershell
docker ps --format "table {{.Names}}\t{{.Status}}" | Select-String "rag_api|vectordb"
docker logs rag_api --tail 20
```

看到 `Application startup complete.`、`Uvicorn running on http://0.0.0.0:8000` 就代表啟動成功。用內部網路測健康檢查：

```powershell
docker exec LibreChat sh -c "wget -qO- http://rag_api:8000/health"
# 應回傳 {"status":"UP"}
```

## 在對話中怎麼正確測試

這是最容易踩坑的地方：**光是把檔案拖進聊天室，預設不會走 RAG。**

1. 點輸入框旁邊的**迴紋針**（不是旁邊那個滑桿「工具」圖示——那個是給網路搜尋/Skills/程式碼執行這些「生成時工具」開關用的，跟這裡要選的東西是兩回事）
2. 選單裡有好幾個選項，**預設第一項是「Upload via Provider」**（把檔案原生塞給模型）——要改選 **「上傳至檔案搜尋」**，這個動作才會把檔案標記成 `tool_resource: file_search`，觸發 `rag_api` 的 embedding
3. 選好檔案後，卡片會顯示處理中，等它跑完（embedding 需要呼叫一次 OpenAI embeddings API，通常幾秒鐘）
4. **送出鈕在輸入框沒有文字時永遠是灰的**，就算檔案已經附加好了也一樣（[SendButton.tsx](../../client/src/components/Chat/Input/SendButton.tsx) 寫死只看文字框內容，不看有沒有附檔）。一定要打字問問題，按鈕才會亮

如果選了「Upload via Provider」（或直接用預設方式）上傳非 PDF/圖片的文件，某些走 OpenAI Responses API 的模型（例如 gpt-5-mini）會直接回錯誤：

```
400 Invalid file data: 'messages[0].content[1].file.file_data'.
Expected a base64-encoded data URL with an application/pdf MIME type ...
but got unsupported MIME type 'text/plain'.
```

原因見 [encodeAndFormatDocuments](../../packages/api/src/files/encode/document.ts)：這個路徑會把檔案編碼成 `input_file` 直接送給模型，OpenAI Responses API 的 `file_data` 只接受 PDF，txt/docx 這類格式會被拒絕。這不是 `rag_api` 的問題，純粹是選錯了附加方式。

### 怎麼確認回答真的有用到 RAG

正常運作時，回答上方會有 **「Searched your files」** 的標籤，展開可以看到具體引用了哪個檔案的第幾段、相似度分數（例如 `59%`）。這是判斷「有沒有真的做檢索」最直接的方式——比起答案讀起來通不通順，這個標籤加上分數才是實際證據。如果問到文件裡沒有的內容，正常的 RAG 回答會老實說「檔案裡查不到」，而不是憑訓練資料編一個內容出來。

## 查詢向量資料庫

### 用 psql（不用裝任何東西）

```powershell
docker exec -it vectordb psql -U myuser -d mydatabase
```

常用查詢：

```sql
-- 看有哪些檔案，各存了幾個 chunk
SELECT e.cmetadata->>'file_id' AS file_id,
       e.cmetadata->>'source' AS source,
       count(*) AS chunks
FROM langchain_pg_embedding e
GROUP BY 1, 2;

-- 看某個檔案實際被切成哪些段落內容
SELECT id, document
FROM langchain_pg_embedding
WHERE cmetadata->>'file_id' = '<file_id>'
ORDER BY id;

-- 看某個 collection 底下有幾筆
SELECT c.name, count(*)
FROM langchain_pg_embedding e
JOIN langchain_pg_collection c ON e.collection_id = c.uuid
GROUP BY c.name;
```

不想開互動式 shell，直接帶 `-c` 也可以：

```powershell
docker exec vectordb psql -U myuser -d mydatabase -c "SELECT count(*) FROM langchain_pg_embedding;"
```

### 用 GUI（DBeaver / TablePlus / pgAdmin）

`vectordb` 預設沒有對外開 port。[docker-compose.override.yml](../../docker-compose.override.yml)（已 gitignore，本機專屬設定，可參考 [docker-compose.override.yml.example](../../docker-compose.override.yml.example)）加了：

```yaml
services:
  vectordb:
    ports:
      - '5433:5432'
```

套用：

```powershell
docker compose up -d vectordb
```

之後用連線字串 `localhost:5433` / 帳號 `myuser` / 密碼 `mypassword` / db `mydatabase` 連線即可。改動只是加 port mapping，容器內資料（volume `pgdata`）不受影響。

## 疑難排解

**上傳檔案後，對話裡出現「RAG API is either not running or not reachable at undefined」**
代表 `RAG_API_URL` 沒有值。檢查 `docker exec LibreChat sh -c "printenv | grep RAG_API_URL"`，本機 compose 應該會自動是 `http://rag_api:8000`。這句錯誤常見於 Railway 這類雲端部署，因為那邊需要另外手動加 `vectordb`/`rag_api` 兩個服務並串好變數引用，不會像本機 compose 一樣自動接好。

**上傳後檔案卡片一直轉圈，送出鈕按不了**
檔案還在 embedding。看 `docker logs rag_api --tail 20`，正常會看到：
```
Starting async pipeline for file ...: N chunks
Generating embeddings for batch ...
HTTP Request: POST https://api.openai.com/v1/embeddings "HTTP/1.1 200 OK"
Async pipeline completed
```
如果卡在這裡沒有動靜，通常是 `OPENAI_API_KEY`（或 `RAG_OPENAI_API_KEY`）失效/額度用完，或該模型的 embeddings API 呼叫逾時。

**想在不碰前端 UI 的情況下直接驗證 `rag_api` 是否正常**
`rag_api` 的 `/embed`、`/query`、`/documents` 都需要帶 LibreChat 簽發的短期 JWT（見 [rag.ts](../../packages/api/src/files/rag.ts) 的 `generateShortLivedToken`），不能單純用 curl 不帶認證打。可以進 `LibreChat` 容器用它自己載入的 `JWT_SECRET` 手動簽一個測試用 token：

```powershell
docker exec LibreChat sh -c "node -e \"require('dotenv').config({path:'/app/.env'}); const jwt=require('jsonwebtoken'); console.log(jwt.sign({id:'<mongo_user_id>'}, process.env.JWT_SECRET, {expiresIn:'5m', algorithm:'HS256'}))\""
```

再用這個 token 呼叫 `POST http://rag_api:8000/embed`（multipart，帶 `file_id` 和 `file`）驗證 embedding，或 `POST /query`（JSON body `{query, file_id, k}`）驗證檢索——這樣可以把「`rag_api` 本身有沒有問題」和「前端有沒有選對附加方式」這兩件事分開驗證，比較好抓根因。

## 參考資料

- [LibreChat 官方文件：RAG API](https://www.librechat.ai/zh/docs/configuration/rag_api)
- [document.ts](../../packages/api/src/files/encode/document.ts) — 檔案原生附加給模型的編碼邏輯（非 RAG 路徑）
- [AttachFileMenu.tsx](../../client/src/components/Chat/Input/Files/AttachFileMenu.tsx) — 附加檔案選單，決定 `tool_resource` 的地方
- [SendButton.tsx](../../client/src/components/Chat/Input/SendButton.tsx) — 送出鈕的啟用條件
- [process.js](../../api/server/services/Files/process.js) — 後端檔案上傳處理，`file_search` 走的分支
- [VectorDB/crud.js](../../api/server/services/Files/VectorDB/crud.js) — 呼叫 `rag_api` `/embed`、`/documents` 的實作
