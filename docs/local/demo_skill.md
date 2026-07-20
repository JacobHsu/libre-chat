# Claude Code `demo-skill`：docx／xlsx／pptx 展示腳本

`.claude/skills/demo-skill/SKILL.md` 是一個 **Claude Code skill**（Claude Code 的 slash command），用來現場操作瀏覽器，展示 LibreChat 的 `docx`／`xlsx`／`pptx` 部署型 skill（見 [skills_docx.md](skills_docx.md)）如何運作。它是給人看操作流程的展示工具，不是自動化測試——驗證正確性請用 [e2e/specs](../../e2e/specs) 底下的 Playwright 測試。

這個 skill 本身依賴 Claude-in-Chrome 瀏覽器擴充功能操作分頁，因此只能在有安裝該擴充功能並連線的 Claude Code 環境中執行；`SKILL.md` 檔案本身不含可執行程式碼。

## 事前準備

- LibreChat 已在 `http://localhost:3080/` 執行（`npm run backend` 或 `docker compose up -d`）。
- 已依 [skills_docx.md](skills_docx.md) 完成 `docx`（以及 `xlsx`／`pptx`，步驟相同）的安裝。
- **`execute_code` 沙箱服務本身正在跑，且是「健康」狀態**——這是與 LibreChat 分開的另一個 `docker compose`
  專案（見 [execute_code.md](execute_code.md)），LibreChat repo 底下的 `docker compose up -d` **不會**連帶啟動它，
  且它可能在 `localhost:3080` 頁面完全正常的情況下處於關閉或只啟動一半的狀態。執行 demo 前先確認：
  ```powershell
  Invoke-WebRequest http://localhost:3112/v1/health
  ```
  回傳非 200 或連不上，代表沙箱沒在跑——不要只看 `docker ps` 覺得有容器在跑就當作沒事，沙箱由六個元件
  組成，只有其中一個健康（例如 `sandbox-runner`）時 API 閘道仍可能連不到，執行到一半才失敗。這一步曾經
  被漏掉：沒先確認沙箱健康就直接展示，結果整段流程（開瀏覽器、開啟工具開關、送出訊息、等待）跑完才在
  最後一步發現失敗，且錯誤看起來像是 skill 本身壞了，其實只是沙箱沒開。

  最常見的情況是「半啟動」：`sandbox-runner` 健康、但 `api`／`redis`／`minio`／`egress_gateway`／
  `file_server`／`tool_call_server` 全部 `Exited (255)`（同一秒一起停，通常是 Docker Desktop／WSL2
  重啟或宿主機睡眠喚醒造成，不是應用程式壞掉；完整判斷方式見 [execute_code.md](execute_code.md#疑難排解)）。
  遇到這種情況，直接補跑即可，不需要重新建置：
  ```powershell
  cd D:\github\code-interpreter
  docker compose -f docker-compose.yaml -f docker-compose.mac.yml up -d
  ```
  跑完等幾秒再重打一次 health check 確認回傳 200，才繼續往下操作瀏覽器。

## 使用方式

- **每個新對話第一次使用前，先手動連線一次瀏覽器擴充功能**：在這個對話裡打一次
  `@browser:new_tab http://localhost:3080/`，等分頁真的開啟後，再下 `/demo-skill xlsx`（或其他格式）。
  這個連線目前觀察到是**綁在單一對話**上、由 `@browser` mention 觸發，`ToolSearch`／`tabs_create_mcp`
  本身無法在從未連線過的對話裡憑空建立連線——如果你直接下 `/demo-skill` 卻沒有先連過，會看到類似
  「Claude-in-Chrome tools aren't available in this environment」的錯誤。


```
/demo-skill docx
/demo-skill xlsx
/demo-skill pptx
/demo-skill            # 等同 all，依序展示三種格式
/demo-skill docx xlsx  # 只展示指定的幾種
```

執行時會依序：

1. 呼叫 `ToolSearch` 載入瀏覽器工具、開啟（或重用）LibreChat 分頁。
2. 若偵測到登入畫面，會停下來請你手動登入，不會自行猜測或代填帳密。
3. **點開輸入框工具選單，確認「Skills」與「執行程式碼」兩個開關都已開啟**，沒開就點開——新開的對話
   （`/c/new`）不會自動沿用之前對話的開關狀態，這兩個沒開，LibreChat 的 `docx`／`xlsx`／`pptx` skill
   就不會真的執行，只會回文字、沒有附件。
4. 針對每個指定格式：在輸入框打一句自然語言需求（例如「幫我用 code interpreter 產生一份簡單的 Word 文件」）、送出、等待回覆完成、確認附件可下載，並錄成一支獨立的 `demo-<格式>.gif`。
5. 最後回報每個格式是否成功產出附件，以及 GIF 的檔名。

拆成每個格式各錄一支 GIF、而非串成一次性長影片，是為了任一步驟失敗時只需重錄該格式，不用整段重來。

## 疑難排解

**`tabs_context_mcp` 或其他瀏覽器工具呼叫失敗**
確認 Claude-in-Chrome 擴充功能仍在瀏覽器中連線；skill 定義中明確要求失敗 2–3 次後停止回報，不會無限重試。

**流程一開始就卡住／頁面顯示登入畫面**
依 skill 定義會直接停下請你手動登入，屬預期行為，手動登入後可重新下指令。

**送出訊息後沒有出現可下載附件**
比照 [skills_docx.md](skills_docx.md) 的疑難排解：確認「執行程式碼」已在工具選單開啟，且對應格式的部署型 skill（`./skill/docx`、`./skill/xlsx`、`./skill/pptx`）已載入成功。

## 參考資料

- [.claude/skills/demo-skill/SKILL.md](../../.claude/skills/demo-skill/SKILL.md) — skill 完整定義
- [skills_docx.md](skills_docx.md) — LibreChat `docx` 部署型 skill 的安裝與使用方式（`xlsx`／`pptx` 步驟相同，僅資料夾名稱不同）
- [execute_code.md](execute_code.md) — `execute_code` 安裝教學，三個檔案格式的 skill 都依賴它才能真正產出檔案
- [e2e/specs](../../e2e/specs) — 需要可重複、可進 CI 的自動化測試時改用這裡
