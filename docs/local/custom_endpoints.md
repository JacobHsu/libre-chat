# 本機開發新增自訂模型提供商

> 官方文件參考：[librechat.ai/zh/docs/quick_start/custom_endpoints](https://www.librechat.ai/zh/docs/quick_start/custom_endpoints)

本文件記錄在這個 repo 的本機 Docker 開發環境（[docker-compose.yml](../../docker-compose.yml)）下，如何新增 OpenAI 相容 / Anthropic 相容的自訂模型提供商（例如 Groq、Mistral、OpenRouter、自架的 vLLM/Ollama 等）。

## 前置知識

- 內建供應商（OpenAI、Anthropic、Google 等）的模型清單是透過 `.env` 裡的 `ANTHROPIC_MODELS`、`GOOGLE_MODELS` 等變數設定（見 [.env.template](../../.env.template#L6)），或不設時由 LibreChat 動態抓取。
- **自訂供應商**不是透過 `.env`，而是透過 `librechat.yaml` 的 `endpoints.custom` 區塊設定。
- 這個 repo 目前**沒有**建立 `librechat.yaml`，且 [docker-compose.yml](../../docker-compose.yml#L30-L37) 也**沒有**掛載它 — 需要靠 `docker-compose.override.yml` 額外掛載，才會被容器讀到。

## 步驟

### 1. 建立 `librechat.yaml`

複製官方範例檔案：

```bash
cp librechat.example.yaml librechat.yaml
```

編輯 `librechat.yaml`，只保留/填寫你要用的 `endpoints.custom` 項目（完整範例見 [librechat.example.yaml:520-558](../../librechat.example.yaml#L520-L558)）。例如新增 OpenRouter：

```yaml
endpoints:
  custom:
    - name: 'OpenRouter'
      apiKey: '${OPENROUTER_KEY}'
      baseURL: 'https://openrouter.ai/api/v1'
      models:
        default: ['meta-llama/llama-3-70b-instruct']
        fetch: true # 設 true 會改成向 API 動態抓可用模型清單
      titleConvo: true
      titleModel: 'meta-llama/llama-3-70b-instruct'
      modelDisplayLabel: 'OpenRouter'
```

若要接 Anthropic 相容的 API（原生 `/v1/messages`），要多加 `provider: 'anthropic'`，寫法參考 [librechat.example.yaml:520-542](../../librechat.example.yaml#L520-L542) 的 `Claude-Compatible` 範例。

### 2. 建立 `docker-compose.override.yml`，掛載設定檔

複製官方範例：

```bash
cp docker-compose.override.yml.example docker-compose.override.yml
```

取消註解「USE LIBRECHAT CONFIG FILE」區塊（[docker-compose.override.yml.example:24-31](../../docker-compose.override.yml.example#L24-L31)），保留 `services:` 與 `api:` 兩層：

```yaml
services:
  api:
    volumes:
    - type: bind
      source: ./librechat.yaml
      target: /app/librechat.yaml
```

### 3. 在 `.env` 加上供應商的 API Key

注意是實際使用的 `.env`，不是 [.env.template](../../.env.template)。例如：

```
OPENROUTER_KEY=sk-xxxx
```

確認 `.env` 的 `ENDPOINTS` 有包含 `custom`（[.env.template:23](../../.env.template#L23) 的預設值已經有）。

### 4. 重啟容器套用設定

```bash
docker compose down
docker compose up -d
```

只是新增 volume 掛載與 `.env` 變數，不需要 `docker compose build`（沒改 image）。

### 5. 驗證

啟動後開啟前端，新的自訂供應商應該會出現在模型選單，和 `ANTHROPIC_MODELS`/`GOOGLE_MODELS` 等內建供應商並列。若沒出現，優先檢查：

- `docker compose logs api` 是否有 `librechat.yaml` 解析錯誤（YAML 縮排、`apiKey` 對應的環境變數是否存在）。
- `docker compose exec api cat /app/librechat.yaml` 確認檔案真的有被掛進容器（步驟 2 的掛載路徑打錯很常見）。
