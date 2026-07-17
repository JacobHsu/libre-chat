# Anthropic「docx」Skill

[Anthropic Agent Skills](https://github.com/anthropics/skills) 是一組給模型使用的技能包，每個技能是一個含 `SKILL.md` 說明文件與輔助腳本的資料夾。`docx` 這個技能教模型如何正確建立、編輯、驗證 Word 文件（`.docx`）。本文件說明如何在這個 LibreChat fork 安裝、啟用並使用它。

`docx` skill 依賴 [`execute_code`](execute_code.md) 才能真正產出檔案；沒有它，模型只能輸出文字內容供手動複製到 Word，等同沒有使用到這個 skill。安裝前建議先確認 `execute_code` 已經可用。

## 事前準備

- LibreChat 以 `docker compose up -d` 啟動
- Git
- 已完成 [execute_code.md](execute_code.md) 的安裝（讓 `docx` 產出的檔案能真正被執行/打包）

## 安裝步驟

### 1. 下載 skill 檔案

```bash
git clone --depth 1 https://github.com/anthropics/skills /tmp/anthropic-skills
mkdir -p ./skill
cp -r /tmp/anthropic-skills/skills/docx ./skill/docx
```

確認檔案結構：

```
skill/
  docx/
    SKILL.md
    scripts/...
```

LibreChat 的 [docker-compose.yml](../../docker-compose.yml) 已經把 `./skill` 掛載進容器（對應 [.env.example](../../.env.example) 的 `DEPLOYMENT_SKILLS_DIR` 設定，預設值就是 `./skill`），不需要額外修改 `.env` 或 `librechat.yaml`。

### 2. 重啟後端

Skills 只在後端啟動時掃描一次，修改 `./skill` 目錄後需要重啟：

```bash
docker compose restart api
```

### 3. 確認載入成功

```powershell
docker compose logs api | Select-String -Pattern "deploymentSkills"
```

預期看到：

```
[deploymentSkills] Loaded 1 deployment skill(s) from /app/skill
```

若看到 `No deployment skills loaded`，確認 `./skill/docx/SKILL.md` 路徑與檔名是否正確。

### 4. 在對話中啟用

點擊聊天輸入框左下角的工具選單（迴紋針右邊的圖示），開啟 **Skills** 與 **執行程式碼** 兩個選項——兩者都要開，缺少「執行程式碼」時模型只能讀到 `docx` 的說明文字，無法真正執行裡面的腳本。

若使用的是 Agent Builder 建立的自訂 Agent，則在該 Agent 設定裡開啟 **Skills**（`skills_enabled`）與 **Run Code**（`execute_code`）兩個開關，效果相同。

## 使用方式

直接描述需求，讓模型自行判斷是否需要呼叫 `docx` skill，例如：

```
請幫我製作一份簡單的 Word (.docx) 文件，標題是「測試報告」，內容隨便寫一段就好。
```

模型會依序顯示 `Ran docx`（載入 skill 說明）、`Writing command`（撰寫執行程式碼）、`Finished running`（執行完成），最後在對話中產出可下載、可預覽的 `.docx` 附件。

> 目前無法透過輸入框的 `$docx` 手動叫用這個 skill（見下方「已知限制」），請一律使用自然語言描述需求。

## 已知限制：`$` 手動叫用目前無法選取部署型 skill

在輸入框打 `$` 叫出的技能清單，目前無法選到透過部署目錄（`./skill`）載入的技能，清單會顯示「No skills yet」。這不影響模型自動判斷叫用（上方「使用方式」），只影響手動叫用這一條路徑。

**原因**：`$` 清單的可見性判斷（[useSkillActiveState.ts](../../client/src/hooks/Skills/useSkillActiveState.ts)）只依「skill 作者是否為目前使用者」決定是否顯示，而部署型 skill 的作者是後端固定的系統帳號（[deployment.ts](../../packages/api/src/skills/deployment.ts) 的 `DEPLOYMENT_AUTHOR_ID`），永遠不等於任何登入使用者，因此被判定為「非本人擁有、預設不啟用」而濾掉。後端在判斷模型是否可自動叫用時（[skills.ts](../../packages/api/src/agents/skills.ts) 的 `resolveSkillActive`）有對部署型 skill 做例外處理，前端目前沒有對應的例外，兩邊行為不一致。

側邊欄「My Skills」列表看得到 `docx`，是因為那個列表單純顯示帳號可存取的 skill，不受此限制影響。

## 疑難排解

**`docker compose logs` 沒有出現 `deploymentSkills` 相關訊息**
確認 `./skill/docx/SKILL.md` 檔案存在，且執行了 `docker compose restart api`（僅修改檔案不會自動重新掃描）。

**模型只回覆文字內容，沒有產出 `.docx` 附件**
確認「執行程式碼」已在輸入框工具選單中開啟，並確認 [execute_code.md](execute_code.md) 的服務已正確部署且可連線。

**`$docx` 在輸入框清單中找不到**
這是目前已知限制（見上方），改用自然語言描述需求即可，不影響模型自動叫用。

## 參考資料

- [LibreChat 官方文件：Skills](https://www.librechat.ai/zh/docs/features/skills)
- [LibreChat 官方文件：Agents](https://www.librechat.ai/zh/docs/features/agents)
- [anthropics/skills：docx](https://github.com/anthropics/skills/tree/main/skills/docx) — 安裝的 `SKILL.md` 與腳本原始碼
- [execute_code.md](execute_code.md) — `execute_code` 的安裝教學（`docx` skill 的前置需求）
- [skill/README.md](../../skill/README.md) — 部署技能目錄說明
- [packages/api/src/skills/deployment.ts](../../packages/api/src/skills/deployment.ts) — 部署技能載入邏輯
- [packages/api/src/agents/skills.ts](../../packages/api/src/agents/skills.ts) — 後端 `resolveSkillActive`
- [client/src/hooks/Skills/useSkillActiveState.ts](../../client/src/hooks/Skills/useSkillActiveState.ts) — 前端 `isActive()` 判斷邏輯
