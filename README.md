<!-- Last synced with README.md: 2026-05-12 (947bfa4c40) -->

<p align="center">
  <a href="https://librechat.ai">
    <img src="client/public/assets/logo.svg" height="256">
  </a>
  <h1 align="center">
    <a href="https://librechat.ai">LibreChat</a>
  </h1>
</p>

<p align="center">
  <a href="README.md">English</a> ·
  <strong>繁體中文</strong>
</p>

<p align="center">
  <a href="https://discord.librechat.ai"> 
    <img
      src="https://img.shields.io/discord/1086345563026489514?label=&logo=discord&style=for-the-badge&logoWidth=20&logoColor=white&labelColor=000000&color=blueviolet">
  </a>
  <a href="https://www.youtube.com/@LibreChat"> 
    <img
      src="https://img.shields.io/badge/YOUTUBE-red.svg?style=for-the-badge&logo=youtube&logoColor=white&labelColor=000000&logoWidth=20">
  </a>
  <a href="https://docs.librechat.ai"> 
    <img
      src="https://img.shields.io/badge/DOCS-blue.svg?style=for-the-badge&logo=read-the-docs&logoColor=white&labelColor=000000&logoWidth=20">
  </a>
  <a aria-label="Sponsors" href="https://github.com/sponsors/danny-avila">
    <img
      src="https://img.shields.io/badge/SPONSORS-brightgreen.svg?style=for-the-badge&logo=github-sponsors&logoColor=white&labelColor=000000&logoWidth=20">
  </a>
</p>

<p align="center">
<a href="https://railway.com/deploy/librechat-official?referralCode=HI9hWz&utm_medium=integration&utm_source=readme&utm_campaign=librechat">
  <img src="https://railway.com/button.svg" alt="Deploy on Railway" height="30">
</a>
<a href="https://zeabur.com/templates/0X2ZY8">
  <img src="https://zeabur.com/button.svg" alt="Deploy on Zeabur" height="30"/>
</a>
<a href="https://template.cloud.sealos.io/deploy?templateName=librechat">
  <img src="https://raw.githubusercontent.com/labring-actions/templates/main/Deploy-on-Sealos.svg" alt="Deploy on Sealos" height="30">
</a>
</p>

<p align="center">
  <a href="https://www.librechat.ai/docs/translation">
    <img 
      src="https://img.shields.io/badge/dynamic/json.svg?style=for-the-badge&color=2096F3&label=locize&query=%24.translatedPercentage&url=https://api.locize.app/badgedata/4cb2598b-ed4d-469c-9b04-2ed531a8cb45&suffix=%+translated" 
      alt="翻譯進度">
  </a>
</p>


# ✨ 功能

- 🖥️ **UI 與體驗**：受 ChatGPT 啟發，並具備更強的設計與功能。

- 🤖 **AI 模型選擇**：
  - Anthropic (Claude), AWS Bedrock, OpenAI, Azure OpenAI, Google, Vertex AI, OpenAI Responses API（包含 Azure）
  - [自訂端點 (Custom Endpoints)](https://www.librechat.ai/docs/quick_start/custom_endpoints)：LibreChat 支援任何相容 OpenAI 規範的 API，無需代理。
  - 相容[本機與遠端 AI 服務供應商](https://www.librechat.ai/docs/configuration/librechat_yaml/ai_endpoints)：
    - Ollama, Groq, Cohere, Mistral AI, Apple MLX, koboldcpp, together.ai,
    - OpenRouter, Helicone, Perplexity, ShuttleAI, DeepSeek, Qwen 等。

- 🔧 **[程式碼解譯器 (Code Interpreter) API](https://www.librechat.ai/docs/features/code_interpreter)**：
  - 安全的沙箱執行環境，支援 Python, Node.js (JS/TS), Go, C/C++, Java, PHP, Rust 與 Fortran。
  - 無縫檔案處理：直接上傳、處理並下載檔案。
  - 隱私無虞：完全隔離且安全的執行環境。

- 🔦 **智能體與工具整合**：
  - **[LibreChat 智能體 (Agents)](https://www.librechat.ai/docs/features/agents)**：
    - 無程式碼自訂助手：無需編程即可建立專業化的 AI 驅動助手。
    - 智能體市集：探索並部署社群建立的智能體。
    - 協作共享：與特定使用者與群組共享智能體。
    - 靈活且可擴充：支援 MCP 伺服器、工具、檔案搜尋、程式碼執行等。
    - [Skills](https://www.librechat.ai/docs/features/skills)：建立可重用的 `SKILL.md` 指令包，用於手動、自動或始終啟用的智能體工作流程。
    - [Subagents](https://www.librechat.ai/docs/features/subagents)：將專門任務委派給擁有獨立上下文視窗的隔離子智能體執行。
    - 相容自訂端點、OpenAI, Azure, Anthropic, AWS Bedrock, Google, Vertex AI, Responses API 等。
    - [支援模型內容協議 (MCP)](https://modelcontextprotocol.io/clients#librechat) 用於工具呼叫。

- 🔍 **網頁搜尋**：
  - 搜尋網際網路並檢索相關資訊以增強 AI 上下文。
  - 結合搜尋供應商、內容爬蟲與結果重排序，確保最佳檢索效果。
  - **可自訂 Jina 重排序**：設定自訂 Jina API URL 用於重排序服務。
  - **[了解更多 →](https://www.librechat.ai/docs/features/web_search)**

- 🪄 **支援程式碼 Artifacts 的生成式 UI**：
  - [程式碼 Artifacts](https://youtu.be/GfTj7O4gmd0?si=WJbdnemZpJzBrJo3) 允許在對話中直接建立 React 元件、HTML 頁面與 Mermaid 圖表。

- 🎨 **圖片生成與編輯**：
  - 使用 [GPT-Image-1](https://www.librechat.ai/docs/features/image_gen#1--openai-image-tools-recommended) 進行文生圖與圖生圖。
  - 支援 [DALL-E (3/2)](https://www.librechat.ai/docs/features/image_gen#2--dalle-legacy), [Stable Diffusion](https://www.librechat.ai/docs/features/image_gen#3--stable-diffusion-local), [Flux](https://www.librechat.ai/docs/features/image_gen#4--flux) 或任何 [MCP 伺服器](https://www.librechat.ai/docs/features/image_gen#5--model-context-protocol-mcp)。
  - 根據提示詞生成驚豔的視覺效果，或透過指令精修現有圖片。

- 💾 **預設與上下文管理**：
  - 建立、儲存並分享自訂預設。
  - 在對話中隨時切換 AI 端點與預設。
  - 編輯、重新提交並透過對話分支繼續訊息。
  - 建立並與特定使用者與群組共享提示詞。
  - [訊息與對話分叉 (Fork)](https://www.librechat.ai/docs/features/fork) 以實現進階上下文控制。

- 💬 **多模態與檔案互動**：
  - 使用 Claude 3, GPT-4.5, GPT-4o, o1, Llama-Vision 與 Gemini 上傳並分析圖片 📸。
  - 支援透過自訂端點、OpenAI, Azure, Anthropic, AWS Bedrock 與 Google 進行檔案對話 🗃️。

- 🌎 **多語言 UI**：
  - English, 中文 (簡體), 中文 (繁體), العربية, Deutsch, Español, Français, Italiano
  - Polski, Português (PT), Português (BR), Русский, 日本語, Svenska, 한국어, Tiếng Việt
  - Türkçe, Nederlands, עברית, Català, Čeština, Dansk, Eesti, فارسی
  - Suomi, Magyar, Հայերեն, Bahasa Indonesia, ქართული, Latviešu, ไทย, ئۇيغۇرچە

- 🧠 **推理 UI**：
  - 针對 DeepSeek-R1 等思維鏈／推理 AI 模型的動態推理 UI。

- 🎨 **可自訂介面**：
  - 可自訂的下拉選單與介面，同時適合進階使用者與初學者。

- 🌊 **[可恢復串流 (Resumable Streams)](https://www.librechat.ai/docs/features/resumable_streams)**：
  - 永不遺失回應：AI 回應在連線中斷後自動重連並繼續。
  - 多分頁與多裝置同步：在多個分頁開啟同一對話，或在另一台裝置上繼續。
  - 生產級可靠性：支援從單機部署到基於 Redis 的水平擴展。

- 🗣️ **語音與音訊**：
  - 透過語音轉文字與文字轉語音實現免持對話。
  - 自動發送並播放音訊。
  - 支援 OpenAI, Azure OpenAI 與 ElevenLabs。

- 📥 **匯入與匯出對話**：
  - 從 LibreChat, ChatGPT, Chatbot UI 匯入對話。
  - 將對話匯出為截圖、Markdown、文字、JSON。

- 🔍 **搜尋與發現**：
  - 搜尋所有訊息與對話。

- 👥 **多使用者與安全存取**：
  - 支援 OAuth2, LDAP 與電子郵件登入的多使用者安全驗證。
  - 內建稽核系統與 Token 消耗管理工具。

- ⚙️ **設定與部署**：
  - 支援代理、反向代理、Docker 及多種部署選項。
  - 使用 [S3 與 CloudFront](https://www.librechat.ai/docs/configuration/cdn/cloudfront) 獲得穩定的媒體連結、邊緣分發、簽名 Cookie 與安全下載。
  - 可完全本機執行或部署在雲端。

- 📖 **開源與社群**：
  - 完全開源且在公眾監督下開發。
  - 社群驅動的開發、支援與回饋。

[查看我們的文件了解更多功能詳情](https://docs.librechat.ai/) 📚

## 🪶 LibreChat：全方位的 AI 對話平台

LibreChat 是一個自托管的 AI 對話平台，在一個注重隱私的統一介面中整合了所有主流 AI 服務供應商。

除了對話功能外，LibreChat 還提供 AI 智能體、模型內容協議 (MCP) 支援、Artifacts、程式碼解譯器、自訂操作、對話搜尋，以及企業級多使用者驗證。

開源、活躍開發中，專為重視 AI 基礎設施自主可控的使用者而打造。

---

## 🌐 資源

**GitHub 儲存庫：**
  - **RAG API:** [github.com/danny-avila/rag_api](https://github.com/danny-avila/rag_api)
  - **網站:** [github.com/LibreChat-AI/librechat.ai](https://github.com/LibreChat-AI/librechat.ai)

**其他：**
  - **官方網站:** [librechat.ai](https://librechat.ai)
  - **說明文件:** [librechat.ai/docs](https://librechat.ai/docs)
  - **部落格:** [librechat.ai/blog](https://librechat.ai/blog)

## 🚀 本地安裝

以下步驟可讓你快速在本機執行 LibreChat，完整說明請參考[本地安裝指南](https://www.librechat.ai/zh/docs/quick_start/local_setup)。

**1. 下載專案**
  - 至 [GitHub 儲存庫](https://github.com/danny-avila/LibreChat)手動下載 ZIP，或使用 Git 指令複製：
    ```bash
    git clone https://github.com/danny-avila/LibreChat.git
    ```

**2. 安裝 Docker**
  - 至 [Docker 官方網站](https://www.docker.com/products/docker-desktop/)下載並安裝 Docker Desktop（建議大多數使用者使用）。
  - 安裝完成後請確認 Docker Desktop 已啟動執行；若尚未啟動，可能需要重新啟動電腦。

**3. 啟動應用程式**
  - 進入專案目錄。
  - 複製 `.env.example` 為 `.env`，並填入必要的環境變數：
    ```bash
    cp .env.example .env
    ```
  - 執行以下指令啟動服務：
    ```bash
    docker compose up -d
    ```
  - 完成後即可於瀏覽器開啟 `http://localhost:3080` 使用 LibreChat。

**4. 啟用 Code Interpreter（選用，`docx`／`xlsx`／`pptx` skill 需要）**
  - Code Interpreter 是獨立部署的沙箱服務（[ClickHouse/code-interpreter](https://github.com/ClickHouse/code-interpreter)），**不包含**在上面第 3 步的 `docker compose up -d` 裡，必須另外 clone 並啟動：
    ```bash
    git clone https://github.com/ClickHouse/code-interpreter
    cd code-interpreter
    docker compose -f docker-compose.yaml -f docker-compose.mac.yml up --build -d
    ```
    （Windows 使用者 clone 完成後，需先修正 `.sh` 檔案的換行符，並額外安裝一次 Python/Node/Bun 執行環境，完整步驟見下方文件。）
  - 確認服務健康：
    ```bash
    curl http://localhost:3112/v1/health
    ```
  - 回到 LibreChat 專案的 `.env`，加入：
    ```
    LIBRECHAT_CODE_BASEURL=http://host.docker.internal:3112/v1
    ```
  - 重啟 LibreChat 讓設定生效：
    ```bash
    docker compose restart api
    ```
  - 完整安裝步驟（換行符修正、語言環境安裝、驗證、疑難排解）見 [docs/local/execute_code.md](docs/local/execute_code.md)。

**5. 其他本機服務啟用教學**
  - 🔍 **RAG（檔案搜尋）**：`rag_api` + `vectordb` 已內建在 `docker-compose.yml`，第 3 步的 `docker compose up -d` 會自動一併啟動，不需額外指令，見 [docs/local/rag.md](docs/local/rag.md)。
  - 📄 **docx／xlsx／pptx Skill**：需先完成上方第 4 步的 Code Interpreter，才能真正產出檔案，安裝步驟見 [docs/local/skills_docx.md](docs/local/skills_docx.md)。
  - 🔌 **自訂端點 / Endpoint 開關**：見 [docs/local/custom_endpoints.md](docs/local/custom_endpoints.md)、[docs/local/endpoints.md](docs/local/endpoints.md)。

更多設定資訊，請參考[環境變數設定](https://www.librechat.ai/docs/configuration/dotenv)、[自訂端點設定](https://www.librechat.ai/docs/quick_start/custom_endpoints)，以及[遠端伺服器進階部署](https://www.librechat.ai/docs/local/docker_override)等文件。若要部署到遠端伺服器（而非本機），可參考 [docs/remote/digitalocean.md](docs/remote/digitalocean.md)。

## 🛡️ 管理面板 (Admin Panel)

LibreChat 提供獨立的瀏覽器管理介面，讓管理員可透過 GUI 管理使用者、角色權限、群組與系統授權，取代手動編輯 `librechat.yaml`。

- **啟用方式**：官方 `docker-compose.yml` 已內建 `admin-panel` 服務，執行 `docker compose up -d` 時會自動啟動，無需額外安裝。
- **必要設定**：於 `.env` 設定 `ADMIN_PANEL_SESSION_SECRET`（至少 32 字元），可透過 `openssl rand -hex 32` 產生。
- **存取網址**：預設為 `http://localhost:3000`（可透過 `ADMIN_PANEL_PORT` 調整連接埠），與主要聊天介面 `http://localhost:3080` 是分開的獨立網址。
- **管理功能**：
  - **使用者**：列出、搜尋與檢視所有帳號。
  - **角色**：建立自訂角色（內建 `USER` / `ADMIN` 之外）並編輯功能權限矩陣。
  - **群組**：建立/刪除群組並管理成員。
  - **系統授權 (System Grants)**：將特定管理權限（如 `manage:users`、`read:usage`）單獨授予使用者、群組或角色，無需給予完整 admin 權限。

**參考文件：**
- [Admin Panel 官方教學](https://www.librechat.ai/zh/docs/features/admin_panel)
- [Access Control 存取控制](https://www.librechat.ai/docs/features/access_control)
- [Interface 物件設定 (librechat.yaml)](https://www.librechat.ai/docs/configuration/librechat_yaml/object_structure/interface)

## 📚 專案文件來源

- 官方文件：[docs.librechat.ai](https://docs.librechat.ai/)
- 專案首頁：[librechat.ai](https://librechat.ai)
- GitHub 儲存庫：[github.com/LibreChat-AI/librechat](https://github.com/LibreChat-AI/librechat)
- RAG API：[github.com/danny-avila/rag_api](https://github.com/danny-avila/rag_api)

---
