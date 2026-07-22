# 本機安裝 TAIDE（繁體中文本地模型）

> 官方 / 參考來源：
> - [taide.tw](https://taide.tw/)（TAIDE 計畫官網）
> - [taide.tw/index/download-model](https://taide.tw/index/download-model)（模型下載入口，導向 Hugging Face）
> - [huggingface.co/taide](https://huggingface.co/taide)（官方 Hugging Face 組織，原始權重，多數為 gated model）
> - [huggingface.co/ZoneTwelve/TAIDE-LX-7B-GGUF](https://huggingface.co/ZoneTwelve/TAIDE-LX-7B-GGUF)（社群轉換的 GGUF 量化版，可直接被 Ollama 執行，非官方帳號）
> - [huggingface.co/audreyt/Gemma-3-TAIDE-12b-Chat-2602-GGUF](https://huggingface.co/audreyt/Gemma-3-TAIDE-12b-Chat-2602-GGUF)（社群轉換的最新版 Gemma 3 12B（2026-02，代號 2602）GGUF 量化版，非官方帳號）
> - [TAIDE 模型授權條款（TAIDE L 類模型社群授權同意書）](https://drive.google.com/file/d/1FcUZjbUH6jr4xoCyaronN_slLgcdhEUd/view?usp=drive_link)

TAIDE（Trustworthy AI Dialogue Engine）是國科會主導、聯合產學研推動的繁體中文大型語言模型計畫，訓練資料強化了台灣用語、文化脈絡與正體中文輸出品質。TAIDE 本身不提供 OpenAI 相容 API，因此接進 LibreChat 的方式是：**先用 Ollama 把模型跑起來，再沿用本 repo 既有的 [Ollama 自訂 endpoint 設定](local_models.md)**，不需要另外改 `librechat.yaml`。

## 1. 選擇模型來源

TAIDE 官方在 Hugging Face 釋出多個版本，授權與可商用性各不相同，下載前務必確認：

| 模型 | 基底 | 釋出時間 | 授權重點 | Hugging Face |
|---|---|---|---|---|
| TAIDE-LX-7B / -Chat | Llama 2 | 2024-04 | 可商用 | [taide/TAIDE-LX-7B](https://huggingface.co/taide/TAIDE-LX-7B) |
| TAIDE-LX-13B | Llama 2 | 2024-04 | 僅限學術研究，不可商用 | huggingface.co/taide（組織頁面搜尋） |
| Llama3-TAIDE-LX-8B-Chat-Alpha1 | Llama 3 | 2024-05～06 | 可商用 | [taide/Llama3-TAIDE-LX-8B-Chat-Alpha1](https://huggingface.co/taide/Llama3-TAIDE-LX-8B-Chat-Alpha1) |
| Llama-3.1-TAIDE-LX-8B-Chat | Llama 3.1 | — | 需另外確認授權條款 | [taide/Llama-3.1-TAIDE-LX-8B-Chat](https://huggingface.co/taide/Llama-3.1-TAIDE-LX-8B-Chat) |
| Gemma-3-TAIDE-12b-Chat | Gemma 3 | 2025-08 | 需另外確認授權條款 | [taide/Gemma-3-TAIDE-12b-Chat](https://huggingface.co/taide/Gemma-3-TAIDE-12b-Chat) |
| Gemma-3-TAIDE-12b-Chat-2602（最新） | Gemma 3 | 2026-02 | 需另外確認授權條款 | [taide/Gemma-3-TAIDE-12b-Chat-2602](https://huggingface.co/taide/Gemma-3-TAIDE-12b-Chat-2602)（加入中期訓練強化台灣知識、修正翻譯腔） |

> 每個模型頁面都有各自的授權同意書連結，商用前請自行讀過條款，這份文件不構成法律意見。

**怎麼選：**

- **本機個人使用/內部測試**（不對外提供商業服務）：選最新的 `Gemma-3-TAIDE-12b-Chat-2602`，品質最好、台灣知識最完整，授權「需另外確認」在這個情境下不構成阻礙。
- **要做商用產品/對外服務**：選明確可商用的兩款之一，其中 `Llama3-TAIDE-LX-8B-Chat-Alpha1`（Llama 3 基底）比 `TAIDE-LX-7B`（Llama 2 基底）新，是可商用選項裡最新的；社群已有現成 GGUF 直接上架在 Ollama 官方 library，不需要 `hf.co:` 前綴：`ollama run cwchang/llama3-taide-lx-8b-chat-alpha1:q8_0`。

**A. 官方原始權重（gated，需申請）**

`taide` 組織底下的模型是 Hugging Face gated model，流程：

1. 註冊/登入 [huggingface.co](https://huggingface.co) 帳號。
2. 到對應模型頁面（如 [taide/Llama-3.1-TAIDE-LX-8B-Chat](https://huggingface.co/taide/Llama-3.1-TAIDE-LX-8B-Chat)），閱讀並同意授權條款（Agree and access repository）。
3. 通過後才能用 `huggingface-cli login`（需 Access Token）下載原始 safetensors 權重，或搭配 transformers/vLLM 直接載入。
4. 原始權重沒有 GGUF 格式，若要餵給 Ollama 需自行用 `llama.cpp` 的 `convert_hf_to_gguf.py` 轉檔並量化，步驟較繁瑣，適合需要原始精度或要自行微調的情境。

**B. 社群 GGUF 量化版（免申請，最快能跑）**

[ZoneTwelve/TAIDE-LX-7B-GGUF](https://huggingface.co/ZoneTwelve/TAIDE-LX-7B-GGUF) 是第三方（非 TAIDE 官方帳號）把 `TAIDE-LX-7B` 轉成 GGUF 並提供多種量化等級：

| 量化等級 | 檔案大小（約） | 適用情境 |
|---|---|---|
| Q2_K | 2.65 GB | 極輕量，品質明顯下降 |
| Q4_K_M | ~4.5 GB | 一般本機使用的甜蜜點（推薦起手） |
| Q5_K_M | ~5.5 GB | 品質更接近原模型 |
| Q6_K | 5.69 GB | 接近全精度 |
| Q8_0 | 7.37 GB | 幾乎無損，檔案較大 |
| F16 | 13.9 GB | 全精度，不建議一般本機使用 |

這個倉庫是公開（non-gated）的，不需要 Hugging Face 帳號即可直接被 Ollama 拉取，本文件以此為主要路徑。

> 這是社群轉檔，不是 TAIDE 官方發布，使用前建議自行核對模型輸出品質；若專案對「官方權重」有硬性要求，改走 A 路徑。

**C. 社群 GGUF 量化版（最新、品質最好：Gemma-3-TAIDE-12b-Chat-2602）**

如果不在意授權條款仍需自行確認（見第 1 節表格），想要目前最新、品質最好的版本，可以用 [audreyt/Gemma-3-TAIDE-12b-Chat-2602-GGUF](https://huggingface.co/audreyt/Gemma-3-TAIDE-12b-Chat-2602-GGUF) —— 這是把官方 2026-02 釋出的 [taide/Gemma-3-TAIDE-12b-Chat-2602](https://huggingface.co/taide/Gemma-3-TAIDE-12b-Chat-2602)（僅提供 safetensors）轉出來的 GGUF，同樣是公開、非官方帳號的社群轉檔：

| 量化等級 | 檔案大小（約） | 適用情境 |
|---|---|---|
| Q2_K | 5.35 GB | 極輕量，品質明顯下降 |
| Q3_K_M | 6.71 GB | 輕量，品質介於 Q2 與 Q4 之間 |
| Q4_K_M | 8.17 GB | 一般本機使用的甜蜜點（推薦起手） |
| Q5_K_M | 9.46 GB | 品質更接近原模型 |
| Q6_K | 10.8 GB | 接近全精度 |
| Q8_0 | 14 GB | 幾乎無損，檔案較大 |
| F16 | 26.4 GB | 全精度，不建議一般本機使用 |

> 12B 模型比 7B/8B 吃更多 VRAM/記憶體，下載前留意本機硬體是否吃得消。舊版（2025-08，[taide/Gemma-3-TAIDE-12b-Chat](https://huggingface.co/taide/Gemma-3-TAIDE-12b-Chat)）的社群 GGUF 見 [nctu6/Gemma-3-TAIDE-12b-Chat-GGUF](https://huggingface.co/nctu6/Gemma-3-TAIDE-12b-Chat-GGUF)，一般沒有理由選舊版。

## 2. 用 Ollama 下載並執行

前提：已依照 [local_models.md 第 1 節](local_models.md#1-安裝-ollama) 裝好 Ollama（主機安裝或容器皆可）。

**主機安裝的 Ollama：**

```powershell
ollama run hf.co/ZoneTwelve/TAIDE-LX-7B-GGUF:Q4_K_M
```

第一次執行會自動下載該量化版本並啟動互動對話，Ctrl+D 或輸入 `/bye` 離開。之後模型已快取在本機，`ollama list` 應該會看到類似 `hf.co/zonetwelve/taide-lx-7b-gguf:q4_k_m` 的項目。

**容器內的 Ollama（compose 服務）：**

```powershell
docker exec -it ollama ollama run hf.co/ZoneTwelve/TAIDE-LX-7B-GGUF:Q4_K_M
```

> 想換其他量化等級，把 `:Q4_K_M` 換成上表的其他 tag（如 `:Q5_K_M`、`:Q8_0`）即可，也可以同時拉多個等級比較品質與速度。

**選用最新版本（Gemma-3-TAIDE-12b-Chat-2602）：**

把上面指令的 repo 換成第 1 節 C 選項的社群 GGUF 即可，其餘步驟相同：

```powershell
ollama run hf.co/audreyt/Gemma-3-TAIDE-12b-Chat-2602-GGUF:Q4_K_M
```

容器版：

```powershell
docker exec -it ollama ollama run hf.co/audreyt/Gemma-3-TAIDE-12b-Chat-2602-GGUF:Q4_K_M
```

## 3. 讓 TAIDE 出現在 LibreChat

本 repo 的 [librechat.yaml:653-662](../../librechat.yaml#L653-L662) 已經設定好 `custom` endpoint「Ollama」，且 `models.fetch: true`，代表**只要模型出現在 `ollama list`，重新整理 LibreChat 前端的模型選單就會自動列出**，不需要額外編輯 `librechat.yaml`。

步驟：

1. 確認第 2 節的 `ollama run` 指令已成功下載並至少執行過一次。
2. 確認 [.env](../../.env) 的 `ENDPOINTS` 有包含 `custom`（本 repo 目前設定見 [local_models.md 第 5 節](local_models.md#5-常見漏踩ENDPOINTS-沒加-custom)，這是最容易漏掉的一步）。
3. 若之前沒開過 Ollama endpoint，重啟 `api` 容器：
   ```powershell
   docker compose restart api
   ```
4. 到前端「Ollama」endpoint 底下的模型清單，應該會看到 `hf.co/zonetwelve/taide-lx-7b-gguf:q4_k_m`（模型 tag 名稱依你下載的版本而定）。

## 4. 驗證與疑難排解

- **模型清單沒出現該 TAIDE 模型**：先在終端機/容器內執行 `ollama list` 確認模型確實下載成功；再檢查 LibreChat 是否連得到 Ollama（見 [local_models.md 第 7 節](local_models.md#7-驗證) 的連線測試指令）。
- **回覆品質不如預期／中文有亂碼**：優先換用較高量化等級（如從 Q4_K_M 換成 Q5_K_M 或 Q8_0）；GGUF 轉檔/量化過程本身可能引入細微品質損失，跟官方 A 路徑的原始權重不完全等價。
- **要跑 13B 版本**：確認你的用途是學術研究（該版本不可商用），且本機 VRAM/記憶體足夠（13B 全精度需求遠高於 7B/8B，建議先查該版本是否已有社群 GGUF 轉檔，否則需自行走 A 路徑轉檔）。

## 補充：跟 Ollama 教學的關係

本文件只補充「TAIDE 模型從哪裡來、怎麼下載」，LibreChat ↔ Ollama 的串接方式（`librechat.yaml` 設定、`baseURL` 對照表、`ENDPOINTS` 常見踩坑）完全沿用 [local_models.md](local_models.md)，不重複說明。
