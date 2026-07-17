# 部署到 DigitalOcean

> 官方文件參考：
> - [librechat.ai/zh/docs/remote/digitalocean](https://www.librechat.ai/zh/docs/remote/digitalocean)（建立 droplet）
> - [librechat.ai/zh/docs/remote/docker_linux](https://www.librechat.ai/zh/docs/remote/docker_linux)（安裝 Docker + 部署 LibreChat）

本文件是把這個 fork（`JacobHsu/libre-chat`）部署到 DigitalOcean droplet 的規劃清單。撰寫時尚未實際跑過整套流程，之後照做時如果遇到跟這裡描述不同的狀況，回來把實際踩到的坑補進「疑難排解」段落，比照 [railway.md](railway.md) 的寫法。

## 為什麼是 DigitalOcean，不是 Railway

[`execute_code`](../local/execute_code.md)（`docx` skill 等功能依賴的程式碼執行沙箱）需要自行部署 [ClickHouse/code-interpreter](https://github.com/ClickHouse/code-interpreter) 這套獨立的 5 容器 `docker compose` 服務，需要能自由控制 `docker compose`（多個獨立 compose 專案、容器間網路），這是 Railway 這種 PaaS 給不了的，所以只能走真正的 VM。DigitalOcean 是官方文件裡最簡單、有 $200 額度可用的選項。

`railway.json`、`.env.railway`、`.env.railway_admin` 仍留在 repo 裡供對照，但目前的部署方向是 DigitalOcean。

## 架構總覽

droplet 上最終會同時跑兩套**各自獨立**的 docker compose 專案：

1. **這個 repo**，用 [deploy-compose.yml](../../deploy-compose.yml)（官方 Docker 遠端指南指定用這個檔案，不是本機開發用的 [docker-compose.yml](../../docker-compose.yml)——服務組成幾乎一樣，差異是 `deploy-compose.yml` 多包一個 `client`（nginx）容器做反向代理，且 mongodb/meilisearch 的 port 預設就不對外開）
   - `api`、`admin-panel`、`client`(nginx)、`mongodb`、`meilisearch`、`vectordb`、`rag_api` — **7 個容器**
2. **code-interpreter**，[docs/local/execute_code.md](../local/execute_code.md) 描述的另一套獨立 compose 專案
   - API、worker sandbox、file server、tool-call server、package-init — **5 個容器**

總共約 **12 個容器**同時在同一台 droplet 上跑。

## Droplet 規格建議

官方 DO 文件建議的最低方案（$6/月，1GB RAM / 1 vCPU）是針對「最基本 LibreChat」（大概只有 `api` + `mongodb` 兩三個容器）估的。這個 fork 實際要跑到 12 個容器，其中 `vectordb`（Postgres）、`meilisearch`、`rag_api`、code-interpreter 的 worker sandbox 都不算輕量，1GB／2GB 很可能不夠、會 OOM。

**建議至少 $24/月方案（4GB RAM / 2 vCPU）起跳**，之後依實際負載觀察要不要再升級（DO 的優點是可以隨時線上升級方案，不用重建）。

## Part 0：防火牆規劃

先想清楚要開哪些 port，Part 1 建立防火牆時直接照這張表設：

| Port | 服務 | 是否對外開放 |
|---|---|---|
| 22 | SSH | 開，建議限制來源 IP |
| 80 / 443 | nginx（`client` 容器，`deploy-compose.yml`） | 開 — 使用者實際入口 |
| 3080 | `api`（`deploy-compose.yml` 沒有直接對外開，是透過 nginx 轉發） | **不對外開** |
| 27017 | `mongodb` | **不對外開**（`deploy-compose.yml` 預設就沒開這個 port） |
| 7700 | `meilisearch` | **不對外開**（同上） |
| 3112 | code-interpreter API | **不對外開**，只綁 `127.0.0.1:3112`，LibreChat 容器透過 `host.docker.internal` 存取（[docker-compose.yml](../../docker-compose.yml) 的 `api` service 已有 `extra_hosts: host.docker.internal:host-gateway`，Linux Docker 20.10+ 同樣支援） |
| 3190 / 2000 / 16379 / 19000 / 19001 | code-interpreter 其他內部服務（egress gateway、sandbox、Redis、MinIO） | **不對外開**，不需要 publish 到 host，留在 code-interpreter 自己的 compose network 即可 |

> **這條特別重要**：[execute_code.md「驗證機制」](../local/execute_code.md)已說明，這個 fork 目前**沒有**對 code-interpreter 做應用層驗證（`LOCAL_MODE=true` 直接放行所有請求）。對外環境完全依賴網路層（防火牆 + 只綁 loopback）保護，port 開錯就等於讓任何人都能對這台 droplet 執行任意程式碼。

## Part 1：建立 Droplet

依 [DigitalOcean 官方文件](https://www.librechat.ai/zh/docs/remote/digitalocean) Part 1：

1. Ubuntu 22.04 LTS、共享 CPU（依上面規格建議選 4GB 方案，而非文件預設的 $6 方案）。
2. 建立非 root 使用者並加入 sudo：`adduser <user>` → `usermod -aG sudo <user>`。
3. 依「Part 0」的表格建立防火牆規則。

## Part 2：安裝 Docker

依官方 [Docker 遠端 Linux 指南](https://www.librechat.ai/zh/docs/remote/docker_linux) 第一部分：

```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-compose-plugin git
sudo usermod -aG docker $USER
sudo reboot
```

重新登入後用 `docker compose version` 確認安裝成功。

## Part 3：部署這個 repo

### 3.1 Clone

```bash
git clone git@github.com:JacobHsu/libre-chat.git
cd libre-chat
```

### 3.2 帶上正式環境 `.env` 與 `librechat.yaml`

`.env` 和 `librechat.yaml` 都沒有進 git（見 [.gitignore](../../.gitignore) 的 `.env*` / `librechat.yaml` 規則），必須另外傳到 droplet，不能只靠 `git clone`：

```powershell
scp .env <user>@<droplet-ip>:~/libre-chat/.env
scp librechat.yaml <user>@<droplet-ip>:~/libre-chat/librechat.yaml
```

傳過去後，正式環境必須修改的值：

| 變數 | 本機值 | 正式環境要改成 |
|---|---|---|
| `DOMAIN_CLIENT` | `http://localhost:3080` | `http://<droplet-ip>`（先用 IP，之後接網域再換成 `https://your.domain`，見 Part 5） |
| `DOMAIN_SERVER` | `http://localhost:3080` | 同上 |
| `HOST` | `localhost` | 不用改，`deploy-compose.yml` 的 `environment` 已強制覆寫成 `0.0.0.0` |
| `MONGO_URI` | `mongodb://127.0.0.1:27017/LibreChat` | 不用改，`deploy-compose.yml` 已覆寫成 `mongodb://mongodb:27017/LibreChat` |
| `LIBRECHAT_CODE_BASEURL` | `http://host.docker.internal:3112/v1` | 不用改，Linux 上 `host.docker.internal` 一樣可用（見 Part 0） |

其餘密鑰（`JWT_SECRET`、`CREDS_KEY`/`CREDS_IV`、`MEILI_MASTER_KEY`、`ADMIN_PANEL_SESSION_SECRET`）可以沿用本機已產生的值，不強制重新產生；但這台 droplet 一旦要正式公開給別人用，安全上建議還是重新跑一次[產生工具](https://www.librechat.ai/toolkit/creds_generator)，不要沿用開發機用過的密鑰。

### 3.3 啟動

```bash
sudo docker compose -f ./deploy-compose.yml up -d
docker ps   # 確認 7 個容器都是 running
```

用 `http://<droplet-ip>` 開啟確認 LibreChat 正常。

## Part 4：部署 code-interpreter（execute_code 依賴）

照 [execute_code.md](../local/execute_code.md) 步驟 1-4，在 droplet 上另外開一個目錄 clone `ClickHouse/code-interpreter`。Linux 原生環境跟文件描述的 Windows 環境有幾個差異：

- **不需要**文件步驟 2 的 CRLF 修正（Linux clone 下來本來就是 LF）。
- **`/dev/kvm`**：一般 DigitalOcean 共享 CPU droplet 沒有 nested virtualization，官方預設的 microVM 隔離模式用不了，一樣要用 `docker-compose.mac.yml`（NsJail-only 模式）——這代表 [execute_code.md「安全性」](../local/execute_code.md)提到的「隔離性弱於官方建議、不要對外公開」在 droplet 上依然成立，這也是 Part 0 防火牆表格要求 code-interpreter 所有 port 都不對外開的原因。
- 步驟 5 設定 port mapping 時，把 code-interpreter API 的對外綁定從預設的 `3112:3112` 改成 `127.0.0.1:3112:3112`（只綁本機 loopback），LibreChat 容器仍透過 `host.docker.internal` 連得到（`host-gateway` 走的是 host 網路，不受 loopback 綁定限制）。
- 正式環境務必把 `CODEAPI_INTERNAL_SERVICE_TOKEN` 換成強密鑰（文件「安全性」段落已提醒本機預設值是共用開發字串）。

## Part 5：（建議）加網域 + TLS

目前 [client/nginx.conf](../../client/nginx.conf) 預設是純 HTTP（`server_name localhost`，443/SSL 區塊整段被註解掉）。如果之後要接自訂網域：

1. 網域 DNS 指到 droplet IP；如果要用 admin panel 的 `admin.<domain>` 子網域，也要多加一筆 DNS record。
2. 準備憑證（Let's Encrypt / Certbot，或用 Cloudflare 代理）。
3. 依 `nginx.conf` 裡的註解，把 Non-SSL 區塊改回 SSL 區塊、`server_name` 換成你的網域、掛上憑證路徑。
4. `.env` 裡 `DOMAIN_CLIENT`/`DOMAIN_SERVER` 換成 `https://your.domain`，`ADMIN_PANEL_SESSION_COOKIE_SECURE` 可改 `true`，`ADMIN_PANEL_URL` 視需要設成 `https://admin.your.domain`。
5. `sudo docker compose -f ./deploy-compose.yml up -d --force-recreate client`

官方文件也提供 Cloudflare／Traefik／NGINX 三種網域設定指南可參考（見 [docker_linux 第四部分](https://www.librechat.ai/zh/docs/remote/docker_linux)）。

## 之後更新

- 主服務：`npm run update:deployed`（= [config/deployed-update.js](../../config/deployed-update.js)，會 `git pull` + 重新拉映像檔 + 重啟；也有 `npm run stop:deployed` / `npm run start:deployed`）。
- code-interpreter：照它自己 repo 的更新方式，不在這個 repo 管轄範圍。

## 尚未涵蓋 / 之後要補

- 還沒實際跑過這套流程，真的部署時如果遇到跟本文件描述不同的狀況（例如 Railway 那份文件記錄過的 port / 網域解析順序問題），回來補在這裡，比照 [railway.md](railway.md) 的「疑難排解」寫法。
- 資料備份（`mongodump`／`uploads`／`images` 目錄）目前沒有自動化，droplet 掛掉或誤刪資料無法復原，之後可以考慮加 DO Volume 快照或簡單的 cron `mongodump` + 上傳到 Spaces。
- droplet reboot 後，`deploy-compose.yml` 的服務都有 `restart: always` 會自動起，但 code-interpreter 的 compose 是否同樣設定，要去它自己的 repo 確認，沒有的話開機後要手動 `docker compose up -d` 補上這一套。
