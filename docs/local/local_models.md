# 本機安裝與串接本地模型（Ollama）

> 官方文件參考：
> - [librechat.ai/blog/2024-03-02_ollama](https://www.librechat.ai/blog/2024-03-02_ollama)（Ollama 安裝 + docker-compose 教學，**沒有中文版**）
> - [librechat.ai/zh/docs/quick_start/custom_endpoints](https://www.librechat.ai/zh/docs/quick_start/custom_endpoints)（`librechat.yaml` 串接教學，有中文版，但**假設 Ollama 服務已經在跑**）
> - [librechat.ai/docs/configuration/librechat_yaml](https://www.librechat.ai/docs/configuration/librechat_yaml)（`endpoints.custom` 完整欄位說明）

官方文件其實有「本地模型」的內容，只是拆成了兩篇：安裝 Ollama 本身的教學是一篇沒有中文翻譯的部落格文章，串接 LibreChat 的教學則在別的頁面而且不重複安裝步驟。這份筆記把兩者合併，並記錄這個 repo 實際踩到的設定坑（見「5. 常見漏踩：`ENDPOINTS` 沒加 `custom`」）。

Ollama 對外提供 OpenAI 相容的 `/v1` API，所以在 LibreChat 眼中它就是一個普通的 [自訂供應商](custom_endpoints.md)，設定方式與接 Groq、OpenRouter 等完全一樣，只差在 `baseURL` 指向本機/容器，且 `apiKey` 可以隨便填一個字串（Ollama 預設不驗證金鑰）。

## 1. 安裝 Ollama

兩種方式擇一：

**A. Ollama App（跑在 Windows/macOS 主機上，非容器）**

到 [ollama.com/download](https://ollama.com/download) 下載安裝，支援 Nvidia、AMD、Apple Metal GPU 加速。裝好後 Ollama 會在主機的 `11434` port 常駐監聽。

**B. 用 Docker 跑進同一個 compose network（跟 LibreChat 其他服務一起管理）**

[docker-compose.override.yml.example:112-124](../../docker-compose.override.yml.example#L112-L124) 已經有現成的範例：

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              capabilities: [compute, utility]
    ports:
      - '11434:11434'
    volumes:
      - ./ollama:/root/.ollama
```

把這段加進本機的 `docker-compose.override.yml`（gitignored，本機專屬設定）。**沒有 Nvidia GPU 的話要整段刪掉 `deploy:` 區塊**，否則容器會因為要求不存在的 GPU 裝置而啟動失敗。

> **怎麼確認自己有沒有 Nvidia GPU（PowerShell）：**
> ```powershell
> # 列出所有顯示卡（含型號、顯存、驅動版本）
> Get-CimInstance Win32_VideoController | Select-Object Name, AdapterRAM, DriverVersion
>
> # 若已裝 Nvidia 驅動，可用這個確認 CUDA 可被容器使用
> nvidia-smi
> ```

## 2. 下載並執行模型

**A 方式**（主機安裝）：直接在終端機下 `ollama pull llama3` 或 `ollama run llama3`。

> **怎麼確認下載成功：**
> ```powershell
> # 列出已下載的模型（最直接）
> ollama list
>
> # 或用 API 查，models 陣列非空即代表有模型
> Invoke-RestMethod http://localhost:11434/api/tags
> ```
> `ollama list`／API 回應裡出現該模型名稱，就代表下載完成、服務也正常在跑。
>
> **`ollama` 指令在終端機找不到？** 安裝程式不一定會把執行檔加進 PATH，常見安裝路徑是 `%LOCALAPPDATA%\Programs\Ollama\ollama.exe`。確認路徑存在後兩個解法擇一：
> ```powershell
> # 暫時：直接用完整路徑呼叫
> & "$env:LOCALAPPDATA\Programs\Ollama\ollama.exe" pull llama3
>
> # 永久：把資料夾加進使用者 PATH（設定後要重開終端機才生效）
> [Environment]::SetEnvironmentVariable('Path', $env:Path + ";$env:LOCALAPPDATA\Programs\Ollama", 'User')
> ```

**B 方式**（容器）：

```powershell
docker compose up -d ollama
docker exec -it ollama ollama pull llama3
```

模型清單見 [Ollama Library](https://ollama.ai/library)，下載前留意模型大小是否適合你的 GPU/記憶體。

## 3. 在 `librechat.yaml` 加自訂 endpoint

前置作業（建立 `librechat.yaml`、掛載進容器）跟接其他自訂供應商完全一樣，步驟見 [custom_endpoints.md](custom_endpoints.md) 的「1. 建立 `librechat.yaml`」「2. 建立 `docker-compose.override.yml`」。這裡只列 Ollama 專屬的部分：

```yaml
endpoints:
  custom:
    - name: 'Ollama'
      apiKey: 'ollama' # Ollama 不驗證金鑰，填任意非空字串即可
      baseURL: 'http://host.docker.internal:11434/v1/' # 見下方 baseURL 對照表
      models:
        default: ['llama3:latest', 'mixtral', 'phi3']
        fetch: true # 改成向 Ollama 動態抓已下載的模型清單
      titleConvo: true
      titleModel: 'current_model'
      modelDisplayLabel: 'Ollama'
```

`baseURL` 依 Ollama 跑在哪裡而不同：

| Ollama 執行方式 | LibreChat api 容器要用的 `baseURL` |
|---|---|
| A：主機安裝的 Ollama App | `http://host.docker.internal:11434/v1/`（Windows/macOS Docker Desktop 原生支援；Linux 需另外在 `api` 服務加 `extra_hosts: ["host.docker.internal:host-gateway"]`） |
| B：docker-compose 裡的 `ollama` 服務 | `http://ollama:11434/v1/`（同一個 compose network，直接用 service name 解析，不需要 `host.docker.internal`） |

## 4. 在 `.env` 確認金鑰變數（可選）

Ollama 不檢查金鑰，`apiKey` 可以直接寫死在 `librechat.yaml` 裡（如上例的 `'ollama'`），不強制要用環境變數。若想比照其他自訂供應商的寫法改成 `apiKey: '${OLLAMA_KEY}'`，記得在 `.env` 加：

```
OLLAMA_KEY=ollama
```

## 5. 常見漏踩：`ENDPOINTS` 沒加 `custom`

這是本 repo 目前實測會踩到的坑，也是最容易被忽略的一步。所有「自訂供應商」（Ollama、Groq、OpenRouter、Mistral、Helicone、Portkey……）都掛在同一個 endpoint 類型 `custom` 底下，必須先確認 `.env` 的 `ENDPOINTS` 有列出 `custom`：

[.env.template:23](../../.env.template#L23) 的預設值有：

```
ENDPOINTS="openAI,agents,assistants,azureOpenAI,google,anthropic,custom"
```

但本 repo 目前實際使用的 [.env:253](../../.env#L253) 是：

```
ENDPOINTS=openAI,agents,google,anthropic
```

**沒有 `custom`。** 這代表就算 `librechat.yaml` 裡已經設定了 Ollama（或其他任何自訂供應商），選單也不會顯示——跟 [endpoints.md](endpoints.md) 記錄的 `ENDPOINTS` 邏輯是同一套機制。要讓 Ollama 出現，必須把 `.env` 改成：

```
ENDPOINTS=openAI,agents,google,anthropic,custom
```

## 6. 重啟套用

```powershell
docker compose down
docker compose up -d
```

只是改 `.env`/`librechat.yaml`/新增 volume 掛載，不需要 `docker compose build`。

## 7. 驗證

- 開啟前端模型選單，確認出現「Ollama」這個 endpoint，且模型清單有抓到（`fetch: true` 時應顯示 `ollama pull` 過的模型）。
- 若選單沒出現，優先檢查第 5 節的 `ENDPOINTS` 設定，再檢查 `docker compose logs api` 有沒有 `librechat.yaml` 解析錯誤。
- 若選單出現但送出訊息後報連線錯誤，檢查 `baseURL` 是否對應正確的執行方式（見第 3 節的對照表），以及：
  ```powershell
  docker exec LibreChat sh -c "wget -qO- http://host.docker.internal:11434/api/tags"
  # 或（B 方式）
  docker exec LibreChat sh -c "wget -qO- http://ollama:11434/api/tags"
  ```
  能回傳已安裝模型的 JSON 清單就代表網路是通的，問題出在 LibreChat 端設定；連不上則是 Ollama 沒起來或 `baseURL` 寫錯。

## 補充：SSRF 例外清單跟這裡無關

`librechat.yaml` 裡有一段專門提到 Ollama 的 `endpoints.allowedAddresses` 註解（[librechat.yaml:318-334](../../librechat.yaml#L318-L334)），容易誤以為是本教學的必要步驟，但那是給 `apiKey`/`baseURL` 設成 `'user_provided'`（讓使用者自己在前端輸入 baseURL）時的 SSRF 白名單。本教學是管理員在 `librechat.yaml` 寫死 `baseURL`，不會經過那道 SSRF 檢查，不需要動這段設定。
