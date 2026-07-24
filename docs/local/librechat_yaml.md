# 修改 `librechat.yaml` 與套用設定

> 官方文件參考：
> - [librechat.ai/docs/configuration/librechat_yaml](https://www.librechat.ai/docs/configuration/librechat_yaml)（完整欄位說明）
> - [librechat.ai/docs/configuration/librechat_yaml/object_structure/interface](https://www.librechat.ai/docs/configuration/librechat_yaml/object_structure/interface)（`interface` 區塊：控制側邊欄／選單各項功能的顯示與否）

## 1. 檔案位置與掛載方式

專案根目錄的 [librechat.yaml](../../librechat.yaml) 透過 [docker-compose.override.yml:6-8](../../docker-compose.override.yml#L6-L8) bind mount 進 `api` 容器（`container_name: LibreChat`，見 [docker-compose.yml:5-6](../../docker-compose.yml#L5-L6)）：

```yaml
services:
  api:
    volumes:
      - type: bind
        source: ./librechat.yaml
        target: /app/librechat.yaml
```

跟 `.env` 一樣，改本機檔案**不需要重新 build image**，但後端只在啟動時讀取一次（[api/server/services/Config/loadCustomConfig.js](../../api/server/services/Config/loadCustomConfig.js) 載入後由 `createAppConfigService` 快取在記憶體中），改完必須重啟才會生效。

## 2. 常見範例：關閉側邊欄的某個頁籤

以「隱藏側邊欄的 Parameters 頁籤」為例，對應 `interface.parameters`（[packages/data-provider/src/config.ts:1294](../../packages/data-provider/src/config.ts#L1294)，預設值 `true`，見同檔 [1387](../../packages/data-provider/src/config.ts#L1387)）：

```yaml
interface:
  parameters: false
```

`interface` 區塊底下還有很多同類型開關（[packages/data-provider/src/config.ts:1282-1384](../../packages/data-provider/src/config.ts#L1282-L1384)），常用的幾個：

| 欄位 | 作用 |
|---|---|
| `modelSelect` | 模型選擇選單 |
| `parameters` | 側邊欄 Parameters 頁籤 |
| `presets` | Presets 功能 |
| `bookmarks` | 對話書籤 |
| `memories` | Memories 功能 |
| `prompts` | Prompts（可為布林，或細分 `use`/`create`/`share`/`public`） |
| `agents` | Agents（同上，可細分權限） |

## 3. 套用設定：重啟

```bash
docker compose restart api
```

或用容器名稱（本 repo 的 `container_name` 是 `LibreChat`）：

```bash
docker restart LibreChat
```

若是直接跑 `npm run backend`／`npm run backend:dev`：停掉現有 process（`Ctrl+C`）後重新執行，nodemon 不會因為 `librechat.yaml` 變更自動重啟。

### 驗證

重啟後重新整理前端頁面，確認對應的 UI 元素（如側邊欄頁籤、選單項目）已依設定顯示／消失。若沒生效，優先檢查：

- `docker compose logs api` 開頭是否有 `Custom config file loaded:`（代表檔案有被讀到）或 YAML 格式錯誤訊息。
- 縮排是否正確——`librechat.yaml` 是 YAML，欄位需在正確的巢狀層級下（例如 `parameters` 必須在 `interface:` 底下，不能頂層）。
- 是否改到了容器內掛載的檔案來源（本 repo 掛載的是根目錄的 `librechat.yaml`，不是 `librechat.example.yaml`）。
