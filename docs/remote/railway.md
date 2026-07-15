# 部署到 Railway

> 官方文件參考：[librechat.ai/zh/docs/remote/railway](https://www.librechat.ai/zh/docs/remote/railway)

本文件記錄把這個 fork（`JacobHsu/libre-chat`）部署到 [Railway](https://railway.app) 的實際步驟，供之後重新部署或排查問題時參考。

## 前置需求

- 已將程式碼 push 到 GitHub（本 repo 的 `origin` 即為部署來源）。
- 一個 Railway 帳號（用 GitHub 登入即可）。

Railway 的 Docker SDK 部署方式跟本機 [docker-compose.yml](../../docker-compose.yml) 不同：Railway 是以「單一服務對應單一 Dockerfile」為單位，MongoDB 等額外服務需要在同一個 Railway 專案內另外加。

## 步驟

### 1. 建立 Railway 專案

1. 到 [railway.app](https://railway.app)，用 GitHub 帳號登入。
2. **New Project → Deploy from GitHub repo**，選擇 `JacobHsu/libre-chat`。
3. Railway 有時仍會因為偵測到 `package.json` 而預選 Nixpacks，需要手動到 **Settings → Build** 把 **Builder** 改成 **Dockerfile**、**Dockerfile Path** 填 `Dockerfile`（對應根目錄的 [Dockerfile](../../Dockerfile)，不是 [Dockerfile.multi](../../Dockerfile.multi)）。這跟專案根目錄的 [railway.json](../../railway.json) 設定一致，手動確認一次之後就不會再跳回 Nixpacks。

### 2. 加 MongoDB 服務

在同一個專案內點 **+ New → Database → Add MongoDB**。Railway 會建立一個 Mongo 服務並自動產生連線字串變數（`MONGO_URL`、`MONGOHOST`、`MONGOPORT`、`MONGOUSER`、`MONGOPASSWORD` 等，實際變數名以 Mongo 服務的 Variables 頁為準）。

**`MONGO_URL` vs `MONGO_PUBLIC_URL`/`MONGO_PRIVATE_URL`**：Mongo 服務的 Variables 頁可能還會多一個 `MONGO_PUBLIC_URL`（走 Railway 公開 TCP Proxy，只有在該服務開啟「Public Networking」後才會真的生效，沒開的話這個變數旁會有黃色警示圖示，代表參照解析不到）。有些模板版本則是叫 `MONGO_PRIVATE_URL`，兩者指的其實是同一種東西——走 Railway 專案內部私網（`xxx.railway.internal`）的連線字串。

由於 LibreChat 服務跟 Mongo 服務在同一個專案內，**直接用 `MONGO_URL` 即可**（不用管有沒有 `MONGO_PUBLIC_URL`/`MONGO_PRIVATE_URL`），它預設就是可用的私網連線字串，不需要額外開公開網路。

### 3. 開啟對外網域

**這一步要先做，再設步驟 4 的 `DOMAIN_CLIENT`/`DOMAIN_SERVER`**，順序顛倒會導致部署後直接崩潰（見下方「疑難排解」）。

到 LibreChat 服務的 **Settings → Networking**，點 **Generate Domain**，確認目標 port 是 `3080`（對應 [Dockerfile](../../Dockerfile) 的 `EXPOSE 3080`）。這一步會讓 Railway 產生該服務專屬的 `RAILWAY_PUBLIC_DOMAIN` 內建變數，之後 `DOMAIN_CLIENT`/`DOMAIN_SERVER` 才解析得到值。

### 4. 設定 LibreChat 服務的環境變數

在 LibreChat 服務（不是 Mongo 服務）的 **Variables** 頁加入：

| 變數 | 值 |
|---|---|
| `HOST` | `0.0.0.0`（[Dockerfile](../../Dockerfile) 已內建 `ENV HOST=0.0.0.0`，可不設） |
| `PORT` | `3080`——**必設**。Railway 會幫服務自動注入一個系統層級的 `PORT`（值不固定，實測過是 `8080`），會蓋過程式碼裡預設的 3080 fallback，導致跟步驟 3 設定的 Networking 目標 port（3080）對不上，變成 502 `Application failed to respond`。見下方「疑難排解」 |
| `MONGO_URI` | 用 Railway 變數參照語法接到 Mongo 服務，例如 `${{MongoDB.MONGO_URL}}`（依實際變數名調整；注意是 `MONGO_URI` 不是 `MONGO_URL`） |
| `CREDS_KEY` / `CREDS_IV` / `JWT_SECRET` / `JWT_REFRESH_SECRET` | 用 [LibreChat 產生工具](https://www.librechat.ai/toolkit/creds_generator) 產生，不可使用預設值或任何文件/範例裡看到的值 |
| `DOMAIN_CLIENT` / `DOMAIN_SERVER` | 填 `https://${{RAILWAY_PUBLIC_DOMAIN}}`；前提是步驟 3 的 Generate Domain 已經做過，否則這個參照會解析成空字串 |
| AI 供應商金鑰（`OPENAI_API_KEY` 等） | 沒有金鑰的先設成 `user_provided`，改由使用者自行在介面輸入 |

#### 複製貼上版本

[`.env.railway`](../../.env.railway) 是整理過、可直接貼進 Variables → **Raw Editor** 的版本（`__REPLACE_ME__` 記得換成自己用產生工具產生的值）。內容分類如下：

- **必填**：`MONGO_URI`、`HOST`、`PORT`、`DOMAIN_CLIENT`/`DOMAIN_SERVER`、四個安全金鑰。
- **AI 供應商金鑰**：預設全部 `user_provided`（伺服器不設金鑰，使用者自行在介面輸入），想直接用自己的金鑰再把值換掉。
- **登入 / 一般設定 / 防濫用 rate limit**：跟 [.env.example](../../.env.example) 的預設值一致，沒有外部依賴，可直接照抄。
- **刻意不包含**：`SEARCH`、`MEILI_*`、`RAG_API_URL` —— 這些依賴 Meilisearch / RAG API 服務，見下方「尚未涵蓋」段落，你的專案還沒加這些服務前設了會導致變數參照解析失敗。
- **刻意不包含**：`ANTHROPIC_MODELS`/`OPENAI_MODELS`/`GOOGLE_MODELS` —— 網路上流傳的機型清單常包含尚未存在或已下架的 ID，不設的話 LibreChat 會動態抓取供應商當下實際可用的機型，比較不會出錯。

#### `.env.template` 與 `.env.railway` 的差異

[`.env.template`](../../.env.template) 是 Railway 官方樣板（[railway.com/deploy/librechat-official](https://railway.com/deploy/librechat-official)）部署後實際產生的環境變數原樣抄錄（密鑰已換成 `__REPLACE_ME__` 佔位符），當作**參考/對照**用，**不要整份直接貼上使用**——裡面有幾處跟這個 fork 的實際部署狀況對不上：

| 變數 | `.env.template`（官方樣板原樣） | `.env.railway`（改過、可直接用） | 原因 |
|---|---|---|---|
| `MONGO_URI` | `${{MongoDB.MONGO_PRIVATE_URL}}` | `${{MongoDB.MONGO_URL}}` | 你的 Mongo 服務 Variables 頁沒有 `MONGO_PRIVATE_URL` 這個變數，只有 `MONGO_URL`，照抄會參照解析失敗 |
| `MEILI_HOST` / `MEILI_MASTER_KEY` / `MEILI_NO_ANALYTICS` / `SEARCH` / `RAG_API_URL` | 有 | 拿掉 | 依賴 Meilisearch、RAG API 服務，這個 fork 還沒加，見「尚未涵蓋」段落 |
| `ANTHROPIC_MODELS` / `OPENAI_MODELS` / `GOOGLE_MODELS` | 有（清單含 `claude-opus-4-7`、`gpt-5.5` 等不存在的機型 ID） | 拿掉 | 不設的話 LibreChat 會動態抓供應商當下實際可用的機型 |
| `DISCORD_CALLBACK_URL` / `GITHUB_CALLBACK_URL` / `GOOGLE_CALLBACK_URL` | 有 | 拿掉 | `ALLOW_SOCIAL_LOGIN="false"` 時沒有實際作用，純備用值 |
| `CONFIG_PATH` | 有（指向社群維護的 `librechat.yaml`） | 拿掉 | 屬於選配，這個 repo 沒有自己的設定檔可比對，先不預設開啟 |

### 5. 部署與確認

第一次建置會執行 `npm ci` 加前端建置，需要幾分鐘。完成後用 Generate Domain 給的網址開啟確認服務正常。

### 之後更新

`git push origin main` 後，Railway 預設會自動偵測到新 commit 並重新部署，不需要額外操作。

## 疑難排解

### `Failed to start server: Invalid URL`

部署後 log 出現：

```
warn: Error parsing DOMAIN_CLIENT for base path: Invalid URL
...
error: Failed to start server: Invalid URL
```

**原因**：步驟順序顛倒——`DOMAIN_CLIENT`/`DOMAIN_SERVER` 設成 `https://${{RAILWAY_PUBLIC_DOMAIN}}`，但這個服務還沒做步驟 3 的 Generate Domain，`RAILWAY_PUBLIC_DOMAIN` 解析成空字串，變成不合法的 `https://`，服務啟動就崩潰。

**修法**：
1. 到 **Settings → Networking** 點 **Generate Domain**（步驟 3）。
2. 回 **Variables** 頁確認 `DOMAIN_CLIENT`/`DOMAIN_SERVER` 旁的黃色警示圖示消失。
3. 到 **Deployments** 分頁確認有沒有自動觸發新的部署（Railway 偵測到被參照的變數變更，通常會自動重新部署）；沒有的話手動點 **Deploy** 或該 deployment 選單裡的 **Redeploy**。
4. 注意是 **Redeploy，不是 Restart**——Restart 只是重啟同一個已啟動的 deployment，不會重新解析變數參照，套用不到新的網域值。

同一批 log 裡其他幾行不是崩潰原因，可以先不管：

- `Config file YAML format is invalid: ENOENT ... librechat.yaml` —— 這個 repo 沒有自己的 `librechat.yaml`，屬於選配，不影響啟動。
- `CHECK_BALANCE ... deprecated` —— 提示未來要改用 `librechat.yaml` 的 `balance` 欄位，現在設 `"false"` 還是有效。
- `RAG API is either not running or not reachable at undefined` —— 因為沒設 `RAG_API_URL`（見「尚未涵蓋」），只影響檔案上傳的 RAG 功能，不影響其他功能。

### Deployment 顯示成功、log 也正常，但開網址是 `502 Application failed to respond`

用 `curl -D -` 打生成的網域會看到：

```
HTTP/1.1 502 Bad Gateway
x-railway-fallback: true
{"status":"error","code":502,"message":"Application failed to respond"}
```

但 Deploy Logs 完全沒有錯誤，甚至看得到：

```
info: Connected to MongoDB
info: Server listening on all interfaces at port 8080. Use http://localhost:8080 to access it
info: Server readiness checks passing.
```

**原因**：服務**沒有崩潰**，只是實際監聽的 port 跟 Networking 設定的目標 port 對不上。Railway 會幫每個服務自動注入系統層級的 `PORT` 變數（值不固定，這次是 `8080`），[api/server/index.js](../../api/server/index.js) 的邏輯是「`PORT` 有設就用它，沒設才 fallback 回 3080」：

```js
const port = isNaN(Number(PORT)) ? 3080 : Number(PORT);
```

但步驟 3（Generate Domain）當時把 Networking 的目標 port 手動填成 `3080`（對照 Dockerfile 的 `EXPOSE 3080`），Railway 的 edge proxy 因此一直往 3080 轉發流量，而 process 實際監聽的是 Railway 自己塞進來的 `8080`——兩邊對不上，所以連得到 container 但拿不到回應。

> `.env.railway` 已在步驟 4 補上 `PORT="3080"`，之後照文件全新部署不會再遇到這個問題；這段記錄是給用舊版 `.env.railway`（沒有這行）已經部署過、現在要排查既有服務的人看的。

**修法（擇一）**：

1. **（推薦）** 到 LibreChat 服務 **Variables**，新增一筆真正的變數 `PORT=3080`（不是參照語法），蓋掉 Railway 自動注入的值，強迫 app 監聽 3080，跟 Networking 已設定的目標 port 對上。存檔後確認有觸發 **Redeploy**（不是 Restart）。這個做法跟本文件其餘步驟（Networking 目標 port = 3080、Dockerfile `EXPOSE 3080`）一致，之後 Railway 重新指派系統 `PORT` 值也不受影響。
2. 到 **Settings → Networking**，把目標 port 從 `3080` 改成 Deploy Logs 裡實際看到的監聽 port（例如這次的 `8080`）。不用改 Variables，但如果 Railway 之後重新指派了不同的系統 `PORT` 值，又要再改一次，比較不穩定。

## 尚未涵蓋

- Meilisearch（搜尋功能）與 RAG API（[docker-compose.yml](../../docker-compose.yml) 裡的 `meilisearch`、`vectordb`、`rag_api` 服務）目前未搬到 Railway，先不裝也能跑基本功能。之後要加的話，可用對應的 Docker image（`getmeili/meilisearch`、`pgvector/pgvector`）在同一個 Railway 專案內以「Add Service → Docker Image」的方式加入，並比照本機 compose 檔設定對應環境變數。
