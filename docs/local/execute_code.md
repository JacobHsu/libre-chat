# Code Interpreter（execute_code）

LibreChat 的「執行程式碼」功能（`execute_code`／`bash_tool`）透過 API 呼叫一個獨立的沙箱服務來執行程式碼。這個服務不是 LibreChat 內建的一部分，需要另外部署。本文件說明如何在本機（Windows + Docker Desktop）自行架設，並把 LibreChat 接上去。

[Anthropic 的 `docx` skill](skills_docx.md) 依賴這個功能才能真正產出 `.docx` 檔案；沒有它，`docx` skill 只能輸出文字內容供手動複製，等同沒有用到這個 skill。

## 事前準備

- Docker Desktop（本機已在跑，Windows/macOS/Linux 皆可）
- Git
- 約 5-10 分鐘（首次建置映像檔）與幾 GB 磁碟空間

## 這個服務是什麼

`execute_code` 背後是 [ClickHouse/code-interpreter](https://github.com/ClickHouse/code-interpreter)，一個開源（Apache 2.0）的沙箱程式碼執行服務。官方文件的說法是「自行部署一個實例，並把 LibreChat 指向它」——沒有公開的免費 API 可以直接呼叫，也沒有付費訂閱可以購買（該服務的代管版本目前未開放新訂閱）。

服務由五個元件組成，用 `docker compose` 一次啟動：

| 元件 | 作用 |
|---|---|
| API | 接收執行請求的 HTTP 閘道 |
| Worker Sandbox | 實際執行程式碼（NsJail 或 libkrun microVM 隔離） |
| File Server | 管理沙箱產生/上傳的檔案（S3 相容儲存） |
| Tool Call Server | 讓沙箱內程式碼可以呼叫其他工具 |
| Package Init | 一次性任務，安裝 Python/Node/Bun/Bash 執行環境 |

## 安裝步驟

### 1. 下載原始碼

```powershell
git clone https://github.com/ClickHouse/code-interpreter
cd code-interpreter
```

### 2. 修正 Windows 換行符問題

這個 repo 沒有針對 shell script 強制使用 LF 換行的設定，Windows 上 `git clone` 預設會把 `.sh` 檔案轉成 CRLF 換行，導致容器啟動時出現 `exec ...: no such file or directory`。clone 完成後執行：

```bash
find . -name "*.sh" -not -path "./.git/*" | while read f; do
  if file "$f" | grep -q CRLF; then
    sed -i 's/\r$//' "$f"
  fi
done
```

### 3. 啟動服務

官方預設的 compose 設定需要 `/dev/kvm`（Linux 虛擬化裝置），Windows／macOS 沒有這個裝置。使用官方提供的 `docker-compose.mac.yml` 覆寫設定改用 NsJail-only 模式：

```powershell
docker compose -f docker-compose.yaml -f docker-compose.mac.yml up --build -d
```

首次執行會建置多個自訂映像檔，需要幾分鐘。啟動後會佔用以下 port：`3112`（API）、`3190`（egress gateway）、`2000`（sandbox）、`16379`（Redis）、`19000`/`19001`（MinIO）。

確認服務正常：

```powershell
docker compose -f docker-compose.yaml -f docker-compose.mac.yml ps
Invoke-WebRequest http://localhost:3112/v1/health
```

所有服務應顯示 `running`/`healthy`，health check 回傳 200。

### 4. 安裝語言執行環境

沙箱本身不含任何程式語言環境，需要另外執行一次性的安裝任務：

```bash
mkdir -p data/pkgs
docker build -f docker/Dockerfile.package-init -t codeapi-package-init .
docker run --rm -v "$(pwd)/data/pkgs:/pkgs" codeapi-package-init
```

這個步驟會編譯 Python、下載 Node.js 與 Bun，需要幾分鐘。完成後 `data/pkgs` 底下會有 `bash/`、`bun/`、`node/`、`python/` 四個資料夾與一個 `.initialized` 標記檔。

> **Windows + Git Bash 使用者**：`-v` 掛載路徑中的絕對路徑容易被 Git Bash 自動轉換成 Windows 路徑而失效，指令會顯示「成功」但實際上沒有寫入任何檔案。若遇到這個狀況，在指令前加上 `MSYS_NO_PATHCONV=1`，並在完成後直接檢查 `data/pkgs` 資料夾內容是否存在，而不只是看指令輸出。

再建立一個小的設定檔，把裝好的語言環境接到沙箱容器實際讀取的路徑（NsJail-only 模式下，官方預設掛載的路徑不會被讀取）：

```yaml
# docker-compose.windows-fix.yml
services:
  sandbox-runner:
    volumes:
      - ./data/pkgs:/pkgs:ro
```

```powershell
docker compose -f docker-compose.yaml -f docker-compose.mac.yml -f docker-compose.windows-fix.yml up -d sandbox-runner
```

確認語言環境已註冊：

```bash
curl http://localhost:2000/api/v2/runtimes
# 應回傳 bash、javascript/typescript（bun）、node、python 共五筆
```

### 5. 設定 LibreChat

在 LibreChat 專案的 `.env` 加入：

```
LIBRECHAT_CODE_BASEURL=http://host.docker.internal:3112/v1
```

`host.docker.internal` 是 Docker Desktop 提供、指向宿主機的網域名稱，讓 LibreChat 的容器可以連到剛剛啟動的 code-interpreter 服務（兩者是各自獨立的 `docker compose` 專案）。本機測試階段不需要設定 `LIBRECHAT_CODE_API_KEY`，原因見下方「驗證機制」。

儲存後重啟 LibreChat：

```powershell
docker compose restart api
```

### 6. 驗證

在 LibreChat 對話框開啟「執行程式碼」與「Skills」（輸入框左下角「工具」選單），輸入：

```
請幫我製作一份簡單的 Word (.docx) 文件，標題是「測試報告」，內容隨便寫一段就好。
```

模型會依序顯示 `Ran docx`、`Writing command`、`Finished running`，最後在對話中產出一個可下載、可預覽的 `.docx` 附件。

## 驗證機制

官方文件提到需要設定 `LIBRECHAT_CODE_API_KEY` 才能通過驗證，但本機部署不需要，原因如下：

**服務端**：code-interpreter 的 [驗證中介層](https://github.com/ClickHouse/code-interpreter/blob/main/service/src/middleware/auth.ts) 在 `LOCAL_MODE=true` 時會直接放行所有請求，完全不檢查任何 Bearer token 或金鑰。官方 compose 設定的預設值就是 `LOCAL_MODE=${LOCAL_MODE:-true}`——本機部署預設就是這個模式。

**LibreChat 端**：這個 fork 目前使用的 `@librechat/agents`（3.2.61）並未實作讀取 `LIBRECHAT_CODE_API_KEY` 或附加驗證 header 的邏輯，執行請求一律不帶任何憑證送出。

兩者相加：本機這個組合下請求本來就不需要驗證，也不會送出驗證。這也代表若之後要把服務部署到會被公開網路存取的環境，不能單純把 `LOCAL_MODE` 關閉、改用官方的 JWT 驗證模式來保護——這個 fork 目前無法提供對應憑證。對外開放時需要用網路層方式保護（防火牆、VPN、IP 白名單），細節見下方「安全性」。

## 疑難排解

**`sandbox-runner` 一直重啟，或 `docker compose ps` 顯示服務名稱是 `sandbox`/`service` 而非 `api`/`sandbox-runner`**
表示套用到了 `docker-compose.local-dev.yml`（需要真正的 `/dev/kvm`）而不是 `docker-compose.mac.yml`。確認啟動指令的 `-f` 參數，並確認步驟 2 的換行符修正已完成。

**執行程式碼時出現 `"bash-5.2.0 runtime is unknown"`（HTTP 400）**
表示步驟 4 的語言環境安裝尚未完成，或 `data/pkgs` 沒有正確掛載到容器的 `/pkgs`。重新執行 `curl http://localhost:2000/api/v2/runtimes` 確認是否回傳五筆語言環境。

**執行程式碼時出現 `CodeAPI request failed: ... returned 401, body: {"error":"API key is required"}`**
這個錯誤來自 `https://api.librechat.ai`（`LIBRECHAT_CODE_BASEURL` 未設定時的預設值），代表 `.env` 裡的 `LIBRECHAT_CODE_BASEURL` 沒有生效，請求打去了官方代管服務而非本機自架的服務。確認 `.env` 設定正確後執行 `docker compose restart api`。

## 安全性

- 本文件採用的 **NsJail-only 隔離模式**（`docker-compose.mac.yml`）隔離性弱於官方建議的 microVM 模式，僅適合本機開發測試，不要用於執行不受信任來源的程式碼，也不要對外公開。
- `CODEAPI_INTERNAL_SERVICE_TOKEN`（服務內部元件之間使用的權杖）本機預設值為共用的開發用字串，正式環境需替換成強密鑰。
- 若計畫將此服務部署到雲端或會被公開網路存取的環境，請先確認網路層防護（防火牆規則、VPN、IP 白名單）到位——見上方「驗證機制」說明此服務目前無法依賴應用層驗證。

## 參考資料

- [LibreChat 官方文件：Code Interpreter](https://www.librechat.ai/zh/docs/features/code_interpreter)
- [ClickHouse/code-interpreter](https://github.com/ClickHouse/code-interpreter) — 原始碼與 README
- [service/src/middleware/auth.ts](https://github.com/ClickHouse/code-interpreter/blob/main/service/src/middleware/auth.ts) — 驗證邏輯
- [docker-compose.mac.yml](https://github.com/ClickHouse/code-interpreter/blob/main/docker-compose.mac.yml) — 無 `/dev/kvm` 環境適用的設定
- [docker/Dockerfile.package-init](https://github.com/ClickHouse/code-interpreter/blob/main/docker/Dockerfile.package-init) — 語言環境安裝腳本
- [skills_docx.md](skills_docx.md) — `docx` skill 安裝與啟用
