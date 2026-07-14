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
3. Railway 會自動偵測根目錄的 [Dockerfile](../../Dockerfile) 並用它建置，不需要額外設定 buildpack。

### 2. 加 MongoDB 服務

在同一個專案內點 **+ New → Database → Add MongoDB**。Railway 會建立一個 Mongo 服務並自動產生連線字串變數（通常叫 `MONGO_URL`，實際變數名以 Mongo 服務的 Variables 頁為準）。

### 3. 設定 LibreChat 服務的環境變數

在 LibreChat 服務（不是 Mongo 服務）的 **Variables** 頁加入：

| 變數 | 值 |
|---|---|
| `HOST` | `0.0.0.0`（[Dockerfile](../../Dockerfile) 已內建 `ENV HOST=0.0.0.0`，可不設） |
| `MONGO_URI` | 用 Railway 變數參照語法接到 Mongo 服務，例如 `${{MongoDB.MONGO_URL}}`（依實際變數名調整） |
| `CREDS_KEY` / `CREDS_IV` / `JWT_SECRET` / `JWT_REFRESH_SECRET` | 用 [LibreChat 產生工具](https://www.librechat.ai/toolkit/creds_generator) 產生，不可使用預設值 |
| `DOMAIN_CLIENT` / `DOMAIN_SERVER` | 先留空部署一次，拿到 Railway 網域後（例如 `https://xxx.up.railway.app`）回來補上，兩者填一樣的值 |
| AI 供應商金鑰（`OPENAI_API_KEY` 等） | 沒有金鑰的先設成 `user_provided`，改由使用者自行在介面輸入 |

### 4. 開啟對外網域

到 LibreChat 服務的 **Settings → Networking**，點 **Generate Domain**，確認目標 port 是 `3080`（對應 [Dockerfile](../../Dockerfile) 的 `EXPOSE 3080`）。拿到網域後回到步驟 3 補上 `DOMAIN_CLIENT` / `DOMAIN_SERVER`，重新部署一次讓變數生效。

### 5. 部署與確認

第一次建置會執行 `npm ci` 加前端建置，需要幾分鐘。完成後用 Generate Domain 給的網址開啟確認服務正常。

### 之後更新

`git push origin main` 後，Railway 預設會自動偵測到新 commit 並重新部署，不需要額外操作。

## 尚未涵蓋

- Meilisearch（搜尋功能）與 RAG API（[docker-compose.yml](../../docker-compose.yml) 裡的 `meilisearch`、`vectordb`、`rag_api` 服務）目前未搬到 Railway，先不裝也能跑基本功能。之後要加的話，可用對應的 Docker image（`getmeili/meilisearch`、`pgvector/pgvector`）在同一個 Railway 專案內以「Add Service → Docker Image」的方式加入，並比照本機 compose 檔設定對應環境變數。
