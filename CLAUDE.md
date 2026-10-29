# TD 235 拍照維修助手 — TAICA 生成式AI 期末專案

> 給 Claude 的專案說明。每次開始工作前先讀完這份文件。
> 目前在**階段一：手動可行性驗證**（第 6 節）。一次只做一個步驟，做完先給 Roger 看結果，等他確認再往下。

---

## 1. 專案概要

- **課程**：TAICA「生成式AI：文字與圖像生成的原理與實務」（政大蔡炎龍老師），期末**個人**專案
- **題目**：TD 235 拍照維修助手。使用者拍一張 Thorens TD 235 唱盤零件或故障處的照片、打一句症狀 → AI 看懂照片 → 到原廠說明書檢索 → 回答可能原因與維修步驟（標頁碼）→ 附上說明書插圖、在照片上標出相關零件
- **作者**：謝承祐（Roger），學號 F74146115，系級一律寫「資訊二」（成大稱「資訊系」，不要寫「資工」）
- **期末專案發想作業**：11/09 23:59 截止（遲交 11/10 16:00）。發想內容在 `docs/proposal/final_proposal.html`，要貼到 NTU COOL 的文字框（HTML）
- **期末專案本身**：老師目前只給了發想作業的說明，**截止日與繳交規格未知**（TODO：Roger 提供後補上）

---

## 2. 與第七週作業 `td235-rag` 的關係（重要）

本專案的「檢索＋回答」核心就是第七週已繳交的作業「TD 235 唱盤技師」：https://github.com/RogerH0711/td235-rag

- **`td235-rag` 在評分完成前一律不要修改**。它的繳交說明和本專案的發想都用 raw 網址引用了它 `main` 分支的截圖，改動會讓助教看到的圖壞掉
- 需要它的檔案時，**複製**到本 repo，不要在那邊改

### 2.1 直接沿用（不用重做）

| 項目 | 內容 |
|---|---|
| 向量庫 | `rag-data.zip`，已在 Google Drive：`https://drive.google.com/uc?export=download&id=1_-jACKgzovK9Pmrdkmq1jNG23Fs_c258`（Roger 的 Drive，檔案 ID 不變）。11 篇章節、一篇一向量、只用英文原文做 embedding |
| Embedding／索引 | `google/embeddinggemma-2`（鎖 revision）、FAISS `IndexFlatIP`、向量正規化；前綴模板 `title: none \| text: {text}`／`task: search result \| query: {question}` |
| LLM | AIsuite＋Groq，`groq:openai/gpt-oss-120b`，`TEMPERATURE = 0.2` |
| 金鑰 | Colab Secrets，名稱 `Groq` |
| 人設與規則 | 第七週 RAG02 第 9 節的 `PERSONA`＋`SYSTEM_PROMPT`（細節控的「TD 235 唱盤技師」；說明書沒寫就說沒寫、標文件編號、技術名詞保留英文、EV 版可能不同、電源安全提醒） |
| 加分功能 | 相似度門檻 0.62（第 15.1 節）、純 LLM 並排對照、頁碼與圖號提示、`normalize_citations()` |
| 評估基準 | 29 題中文練習問題：top-1 22/29、top-3 27/29（`td235-rag/docs/retrieval-eval.md`） |

固定套件版本（第七週老師規定，期末先沿用，除非 Roger 說可以換）：

```python
%pip install -q "sentence-transformers==6.1.0" "transformers==5.20.1" "faiss-cpu==1.15.1" "aisuite[groq]==0.2.0" "groq==0.37.1" "gradio==6.30.0"
```

### 2.2 說明書的事實（第七週逐頁看過）

- 15 頁掃描 PDF、沒有文字層；英法德三語：**奇數頁 3–11 左英右法、偶數頁德文＋插圖**、p13 全寬英文規格表
- **插圖都在德文偶數頁**：Fig. 1–4 → p4、Fig. 5–6 → p6、Fig. 7–9 → p8、Fig. 10–11 → p10
- 每篇文件的 metadata 有 `figures` 欄位（例如 `Fig. 10 (p.10)`），可以直接對應到要顯示的頁面圖
- 說明書提到的零件編號：lift-lever (5)、collar-lock (9) 等（詳見 `td235-rag/CLAUDE.md` 第 5 節）
- 說明書寫的是 **TD 235**，Roger 的唱盤是 **TD 235 EV**，回答時不可假設完全相同
- 說明書**沒有**轉速微調的說明（Roger 的唱盤實際遇到轉速偏慢）

### 2.3 需要從別處搬過來的

- **頁面圖**：第七週 `tools/render_pages.py` 轉出的 `ocr/pages/pNN_full.png`，**只存在 ROG 上**（沒有進 git）。期末要用 p4、p6、p8、p10（有插圖的德文頁），需要時請 Roger 從 ROG 複製過來，或用 `scp -r rog:TAICA/RAG/ocr/pages ...`
- **RAG02.ipynb**：之後要當成期末 notebook 的起點，從 `td235-rag` 複製一份，不要在原 repo 改

---

## 3. 規則（沿用第七週，期末也照做）

> 期末專案的評分規格還沒公布。以下是第七週的規則，**先當作期末也適用**；Roger 拿到規格後要回來更新這一節。

1. **Colab 連結會公開**：金鑰只能用 Colab Secrets，程式碼、cell 輸出、上傳的檔案都不能出現金鑰
2. **助教要能「全部執行」一次跑完**：不能有 `files.upload()` 這類手動步驟；不可依賴 Roger 的 Drive 掛載、本機路徑或本機 Ollama
3. **註三：用 AI 的地方要附 Prompt 與結果截圖，加上 Roger 理解後的說明**，否則視為抄襲（該次 0 分、總成績 -10）。維護 `docs/ai-usage-log.md`，到關鍵節點主動提醒 Roger 截圖
4. **成大端**：除了政大要求的項目，另外上傳 `.ipynb`
5. 對沿用程式的修改標「🔧 與第七週不同：…」，新增的 cell 標「🔧 新增：…」；每個主要步驟前面放繁體中文 Markdown 說明

---

## 4. 系統流程（發想作業的版本）

1. **使用者輸入**：照片＋一句中文症狀
2. **看圖理解**：Vision LLM 描述照片，固定輸出 JSON `{part, observation, confidence}`
3. **查詢改寫**：照片描述＋症狀 → 英文檢索句（向量庫只收英文原文；是否真的需要，用第 6 節步驟 3 的數據決定）
4. **RAG 檢索**：沿用第七週，Top-3＋門檻 0.62
5. **技師回答**：沿用第七週人設與規則，改成多輪（`gr.Chatbot`）；**檢索只用當次問題＋照片描述**，歷史對話只放進 messages
6. **圖像輸出**：(a) 依 `figures` 顯示插圖所在頁；(b) PIL 在照片上畫框；(c) 選做：文生圖示意插圖，一律標「AI 示意，非原廠圖」

**手動備案**：②的 confidence 低於門檻時，讓使用者從下拉選單選零件名稱，再接 ③。

---

## 5. 目錄結構（預定）

```
td235-photo-assistant/
├── CLAUDE.md
├── README.md
├── docs/
│   ├── proposal/              # 發想作業（HTML＋兩張概念圖），已用 raw 網址引用，不要改檔名
│   ├── feasibility.md         # 階段一的驗證紀錄
│   ├── ai-usage-log.md        # 註三
│   └── screenshots/
├── photos/                    # Roger 拍的測試照片（推上 GitHub 前先確認沒有個人資訊）
├── testset/                   # 照片測試集：照片檔名、症狀、正確章節 doc_id
├── pages/                     # 說明書頁面圖 p4/p6/p8/p10
└── (之後) photo_assistant.ipynb
```

---

## 6. 階段一：手動可行性驗證（目前在這裡）

目的：在寫完整程式之前，確認每一段都走得通。結果寫進 `docs/feasibility.md`，每一步都記錄「做了什麼、結果、結論」。

**步驟 0：確認可用的 vision model**
- 條件：能從 **Colab** 透過 API 呼叫（助教要能跑），最好能用 AIsuite，最好有免費額度
- **以官方文件的實際內容為準，不要憑記憶假設**哪個供應商有哪個模型。先查 Groq 目前有沒有支援圖片輸入的模型、AIsuite 0.2.0 怎麼傳圖片
- 列 2–3 個候選給 Roger 選，寫清楚每個的呼叫方式、限制、費用

**步驟 1：拍照**（Roger 做）
- 自己的唱盤拍約 10 張：皮帶、馬達 pulley、唱臂、配重、升降桿、轉速旋鈕、RCA 端子、唱頭等
- 每張記下「這是什麼零件」「對應說明書哪一章（doc_id）」，存成 `testset/photos.csv`

**步驟 2：看圖測試**
- 同一組照片丟給候選模型，記錄能否說出正確零件、JSON 格式是否穩定
- 先用網頁或最少量的程式手動測，不急著寫完整 pipeline

**步驟 3：檢索測試＋中英文比較**
- 把步驟 2 的描述分成「中文」「英文」兩種寫法，丟進第七週的檢索，算 Top-3 命中率
- 用數據決定要不要做 ③ 查詢改寫

**步驟 4：標註測試**
- 確認模型能不能給出零件在照片中的大概位置（bounding box）
- 不準就改成文字指位（例：「左上角的圓形旋鈕」），不影響主流程

每一步做完都停下來，給 Roger 看結果、提醒要不要截圖（註三），等他確認再做下一步。

---

## 7. 時程（暫定，期末截止日公布後調整）

| 日期 | 目標 |
|---|---|
| 11/09 前 | 發想作業繳交 |
| 11/03–11/16 | 階段一：手動可行性驗證，選定 vision model |
| 11/17–11/30 | 串接 ①–⑤，Gradio 雛形（含多輪追問） |
| 12/01–12/14 | 圖像輸出、照片測試集評估 |
| 之後 | 報告與展示 |

---

## 8. 工作環境

- Roger 有兩台電腦：Mac（主要開發，repo 放在 `~/code/`）與 ROG（Windows，RTX 3070，有 Ollama 和 pdf2md）
- 在 VS Code 編輯，透過 Colab CLI 執行；**Colab CLI 讀不到 Colab Secrets**，需要 `Groq` 的部分只能在 Colab 網頁上跑（詳見 `td235-rag/HANDOFF.md` 第四節）
- commit 前確認沒有金鑰；照片推上 GitHub 前確認沒有個人資訊
