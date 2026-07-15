# 部署 Admin Panel 到 Railway

> 前置文件：先完成 [railway.md](railway.md) 的 LibreChat 主服務部署，本文件只補「加一個 Admin Panel 服務」的部分。

本文件記錄把 LibreChat 官方的 **Admin Panel** 加進同一個 Railway 專案（`librechat-demo`）的實際步驟。目前這個 fork 的 [railway.json](../../railway.json) 只建置了 LibreChat 主服務（讀 repo 自己的 [Dockerfile](../../Dockerfile)），Admin Panel 完全沒有搬過去，所以 Railway 專案畫布上目前只有 `librechat-demo` 跟 `MongoDB` 兩個方塊。

## 架構：Admin Panel 怎麼操作 MongoDB

`librechat-admin-panel` 的環境變數裡**沒有 `MONGO_URI`，也沒有任何資料庫連線字串**，它不會直接連 MongoDB。它是透過 HTTP 打 `librechat-demo`（LibreChat 主服務）的 `/api/admin/*` 系列 API，由 `librechat-demo` 代為讀寫資料庫：

```
瀏覽器 → librechat-admin-panel（前端 UI + 薄後端）
              │ HTTP 打 API_SERVER_URL 指到的 /api/admin/* 路由
              ▼
       librechat-demo（真正連 MongoDB 的服務，有 MONGO_URI）
              │ Mongoose
              ▼
           MongoDB
```

對應的後端路由在 [api/server/routes/admin/](../../api/server/routes/admin/)：

| 檔案 | 功能 |
|---|---|
| `auth.js` | 登入 / OAuth（OpenID、SAML、Google、GitHub、Discord、Facebook、Apple） |
| `users.js` | 使用者列表 / 搜尋 |
| `roles.js` | 角色管理 |
| `groups.js` | 群組管理 |
| `grants.js` | 權限授予 |
| `config.js` | 系統設定 |
| `audit.js` | 稽核紀錄 |
| `skills.js` | 技能管理 |

以 [api/server/routes/admin/users.js:14-25](../../api/server/routes/admin/users.js#L14-L25) 為例，`listUsers`／`searchUsers` 呼叫的 `db.findUsers`／`db.countUsers`（來自 `~/models`，底層是 Mongoose）都是跑在 `librechat-demo` 這個 container 裡，且每個路由都會用 `requireJwtAuth` + `requireCapability(ACCESS_ADMIN)` 檢查呼叫者是不是有效的管理員帳號才放行。

這也解釋了為什麼步驟 2 的 Variables 裡只有 `API_SERVER_URL`（指到 `librechat-demo` 私網），沒有也不需要 `MONGO_URI`。

## 先搞懂本地端在跑什麼

本地 [docker-compose.yml](../../docker-compose.yml) 裡其實有三個 service，`admin-panel` 是其中之一：

```yaml
services:
  api:
    image: registry.librechat.ai/danny-avila/librechat-dev:latest   # 官方預建 image，不是本地 build
  admin-panel:
    image: registry.librechat.ai/clickhouse/librechat-admin-panel:latest  # 官方預建 image
    ports:
      - "${ADMIN_PANEL_PORT:-3000}:3000"
    environment:
      - PORT=3000
      - SESSION_SECRET=${ADMIN_PANEL_SESSION_SECRET}
      - API_SERVER_URL=http://api:${PORT:-3080}
      - VITE_API_BASE_URL=${DOMAIN_CLIENT:-http://localhost:3080}
      - SESSION_COOKIE_SECURE=${ADMIN_PANEL_SESSION_COOKIE_SECURE:-false}
  mongodb: ...
```

**重點**：整份 `docker-compose.yml` 沒有任何 `build:` 設定（用 `grep '^\s*build:' docker-compose.yml` 確認過），所以 `docker compose up` 抓的兩個 image（`api`、`admin-panel`）都是官方預建好的，**跟你本地改的原始碼無關**。要讓本地程式碼改動生效，開發時請用 `npm run backend:dev` + `npm run frontend:dev`（見 [CLAUDE.md](../../CLAUDE.md)），不要靠 `docker compose up` 驗證改動。

反過來，Railway 上的 `librechat-demo` 服務**是**用你 repo 的 [Dockerfile](../../Dockerfile) 現場 build（[railway.json](../../railway.json) 的 `"builder": "DOCKERFILE"`），這點跟本地 docker-compose 不一樣。

Admin Panel 本身在本地跟 Railway 上都是官方現成 image，這件事在兩邊是一致的，只需要照抄 `docker-compose.yml` 的環境變數設定，換成 Railway 的變數參照語法即可。

## 步驟

### 1. 新增 Admin Panel 服務

在 `librechat-demo` 專案內（跟當初加 MongoDB 同一個「+ New」入口）：

1. **+ New → Docker Image**（有些版本要選 **Empty Service**，建立後再到 Settings → Source 手動填 image）。
2. Image 欄位填：
   ```
   ghcr.io/clickhouse/librechat-admin-panel:latest
   ```

   ⚠️ **不要填 `docker-compose.yml` 裡寫的 `registry.librechat.ai/clickhouse/librechat-admin-panel:latest`**——這個 mirror 網域會把身分驗證跨網域轉到 `ghcr.io/token`（用 `curl` 查 registry API 確認過，兩邊指向同一個 image、tags 完全一樣），但 Railway 的 Docker Image 欄位驗證處理不了這種跨網域 redirect，會顯示 `Invalid Docker image` 拒絕輸入。實測直接填 `ghcr.io` 的路徑就正常了。

建立後畫布上會多一個獨立方塊，跟 `librechat-demo`、`MongoDB` 平行，不是疊在既有服務上。

### 2. 設定 Admin Panel 服務的環境變數

對照 `docker-compose.yml` 的 `admin-panel` 區塊，在**這個新服務**的 Variables 頁加入：

| 變數 | 值 | 說明 |
|---|---|---|
| `PORT` | `3000` | **必設**。原因跟主服務一樣——Railway 會自動注入系統層級的 `PORT`（值不固定），不手動設會蓋過官方 image 預期的 3000，導致跟 Networking 目標 port 對不上，變成 502。見 [railway.md 的疑難排解](railway.md#deployment-顯示成功log-也正常但開網址是-502-application-failed-to-respond) |
| `SESSION_SECRET` | 自己產生一組隨機字串，或沿用本地 `.env` 裡的 `ADMIN_PANEL_SESSION_SECRET` 值 | image 沒這個變數會拒絕啟動（見 `docker-compose.yml` 註解） |
| `API_SERVER_URL` | `http://${{librechat-demo.RAILWAY_PRIVATE_DOMAIN}}:3080`（服務名稱依你在 Railway 上實際命名調整，通常是你的 GitHub repo 服務名） | 對照本地的 `http://api:3080`——用 Railway 私網域名 + LibreChat 主服務監聽的 port（依 [railway.md](railway.md) 設定應該是 `3080`） |
| `VITE_API_BASE_URL` | `${{librechat-demo.DOMAIN_CLIENT}}`（或直接 `https://${{librechat-demo.RAILWAY_PUBLIC_DOMAIN}}`） | 對照本地的 `DOMAIN_CLIENT`——瀏覽器（不是 container）要連到的 LibreChat API 公開網址 |
| `SESSION_COOKIE_SECURE` | `true` | Railway 上是 HTTPS，正式環境建議開啟；本地預設 `false` 是因為本地是 HTTP |

#### 複製貼上版本

[`.env.railway_admin`](../../.env.railway_admin) 是整理過、可直接貼進這個新服務的 Variables → **Raw Editor** 的版本（`SESSION_SECRET` 的 `__REPLACE_ME__` 記得換成自己產生的隨機字串，或沿用本地 `.env` 裡的 `ADMIN_PANEL_SESSION_SECRET` 值）。`API_SERVER_URL`/`VITE_API_BASE_URL` 裡的 `librechat-demo` 是這個 fork 目前 Railway 專案裡主服務的名稱，如果你的服務名稱不同要跟著改。

⚠️ **`VITE_API_BASE_URL` 未實測**：Vite 專案的環境變數正常是 **build-time** 寫死進靜態檔案的，官方這個 image 是否有做 runtime envsubst / 替換佔位符，這份文件目前沒有驗證過。如果照這個設定部署後，Admin Panel 頁面打的 API 網址不對（例如還是打 `localhost:3080`），代表這個 image 不支援 runtime 注入，需要回報或另尋官方文件確認正確用法。

### 3. 開啟對外網域

Admin Panel 服務的 **Settings → Networking → Generate Domain**，目標 port 填 `3000`（對照步驟 2 設定的 `PORT`）。

### 4. 回頭設定 LibreChat 主服務

回到 `librechat-demo` 服務的 Variables，新增：

| 變數 | 值 | 說明 |
|---|---|---|
| `ADMIN_PANEL_URL` | Admin Panel 服務剛生成的公開網址，例如 `https://${{librechat-admin-panel.RAILWAY_PUBLIC_DOMAIN}}`（服務名稱依實際建立時取的名字調整，這個 fork 的 Railway 專案裡是 `librechat-admin-panel`） | [api/server/routes/admin/auth.js](../../api/server/routes/admin/auth.js) 的 OAuth 相關 redirect（`/api/admin/oauth/*`）都靠這個變數組 callback URL。不設的話會 fallback 回 [packages/api/src/auth/exchange.ts](../../packages/api/src/auth/exchange.ts) 定義的 `http://localhost:3000`，正式站上一定連不到 |

存檔後確認 `librechat-demo` 有觸發 **Redeploy**（不是 Restart），這樣新的 `ADMIN_PANEL_URL` 才會生效。

### 5. 確認

用步驟 3 生成的網址開 Admin Panel，應該會看到登入畫面。能登入的帳號必須具備 `ACCESS_ADMIN` 能力（一般對應 `ADMIN` 角色）——這份文件還沒涵蓋「怎麼把一個帳號升級成 ADMIN」，目前這個 fork 的 `config/` 底下（`create-user.js`、`list-users.js` 等）沒有現成的角色升級腳本，需要另外確認（可能要直接改 MongoDB 的 `users` collection，或透過 LibreChat 既有的角色管理機制）。

## 疑難排解

### Docker Image 欄位顯示 `Invalid Docker image`

貼上 `registry.librechat.ai/clickhouse/librechat-admin-panel:latest` 後，Railway 顯示紅字 `Invalid Docker image`，即使確定字串完整、沒有截斷。

**原因**：`registry.librechat.ai` 是轉包到 `ghcr.io` 的 mirror，身分驗證的 `www-authenticate` 會指向另一個網域（`ghcr.io/token`）。Railway 的 Docker Image 輸入框驗證器處理不了這種跨網域的 auth realm redirect，即使圖片本身完全有效（用 `curl` 直接查 Docker Registry API v2 確認過，`registry.librechat.ai` 跟 `ghcr.io` 兩邊回傳一樣的 tag 清單，都有 `latest`）。

**修法**：改填 ghcr.io 的原始路徑，見步驟 1。

### 502 `Application failed to respond`

跟 [railway.md](railway.md) 記錄過的主服務問題成因一樣：Admin Panel 服務實際監聽的 port 跟 Networking 目標 port 對不上。檢查步驟 2 有沒有確實設 `PORT=3000`，並確認 Deploy Logs 裡看到的監聽 port 是不是真的是 `3000`。

### Admin Panel 頁面打不到 API / CORS 錯誤

多半是 `API_SERVER_URL`（container 對 container，要用私網域名）或 `VITE_API_BASE_URL`（瀏覽器對外，要用公開網域名）兩者填反或填錯。這兩個變數的對象不一樣：前者是 Admin Panel **後端**連 LibreChat API 用的，後者是 Admin Panel **前端**（瀏覽器執行的 JS）連 LibreChat API 用的。

### OAuth 登入完成後導回 `localhost:3000`

代表步驟 4 的 `ADMIN_PANEL_URL` 沒設定或沒生效，檢查 `librechat-demo` 服務的 Variables 並確認已 Redeploy。

## 尚未涵蓋

- 怎麼把一個既有帳號的角色升級成 `ADMIN`（`ACCESS_ADMIN` 能力）。
- `VITE_API_BASE_URL` 在官方 image 裡是否支援 runtime 注入，目前只是照抄本地 `docker-compose.yml` 的設定推測，沒有在 Railway 上實際跑通驗證過。
- Admin Panel 支援的 OAuth 提供者（OpenID／SAML／Google／GitHub／Discord／Facebook／Apple）在 Railway 環境下的完整設定，本文件只涵蓋讓服務跑起來、能連到 API 的部分。
