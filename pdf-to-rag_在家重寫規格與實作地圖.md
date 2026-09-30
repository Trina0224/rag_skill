# PDF-to-RAG 在家重寫規格與實作地圖

> 依據 `C:\Users\trinshih\Documents\fae-skills-hub\skills\pdf-to-rag` 於 2026-09-30 的程式、文件及測試整理。這是一份供你在個人電腦**重新實作**的設計文件，沒有附帶原始程式、測試 PDF、金鑰或 `.env` 內容。本文描述的是當時原版的實際行為；標記「重寫建議」的項目是改良方向，並非原版已具備的功能。

## 0. 先看結論

這個 skill 的核心是兩條 CLI 流程：`ingest.py` 把一份具有文字層的 PDF 轉成獨立 FAISS store；`query.py` 對該 store 做向量與 BM25 混合檢索、RRF 融合、LLM 重排，最後要求 LLM 只依檢索片段回答並標頁碼。每份 PDF 一個 store，不做索引合併。掃描影像 PDF 沒有 OCR；圖表中的像素資訊靠查詢後按頁渲染人工查看。

AMD Gateway 不是 RAG 演算法的必要部分。它在目前程式中同時提供**聊天模型**與 **embedding 模型**，因此在家重寫時應把這兩種服務拆成明確介面，再接你自己的雲端或本機後端。任何 embedding 模型的更換，都必須重新建索引；同一個 store 的文件與查詢必須用同一個模型及相同前處理。

## 1. 盤點範圍與檔案地圖

檢查範圍包含 `SKILL.md`、`README.md`、`REFERENCE.md`、`DESCRIPTION.md`、`EXAMPLES.md`、`AGENTS.md`、`requirements.txt`、`pytest.ini`、`scripts/` 中 12 個 `.py`、`tests/` 中 8 個 `test_*.py` 與 `conftest.py`。另外發現 `.env`、測試 PDF、`.pytest_cache/`、`__pycache__/`；它們不是重寫規格的來源，尤其未讀取 `.env` 值。`EXAMPLES.md` 的公開論文與 AMD 文件敘述只用來確認操作情境，無須攜帶那些 PDF。

| 檔案 | 在系統中的職責 | 重寫時的優先度 |
|---|---|---|
| `scripts/ingest.py` | CLI、參數驗證、完整匯入編排、續跑、輸出組裝 | 最高 |
| `scripts/query.py` | 載入 store、檢索、回答、文字與 JSON 輸出 | 最高 |
| `scripts/pdf_extract.py` | pypdf 抽字、逐頁資料、重複行移除、Symbol 字元修正 | 高 |
| `scripts/quality_check.py` | PDF 是否有足夠可擷取文字 | 高 |
| `scripts/llm_review.py` | 以有限預覽偵測標題、種類、摘要、章節起始頁 | 高 |
| `scripts/chunking.py` | 章節邊界、短頁合併、長文重疊切塊 | 高 |
| `scripts/summarize.py` | 每塊產生帶文件脈絡的簡述 | 中 |
| `scripts/embedding.py` | embedding provider 介面、批次化、逐批 checkpoint | 高 |
| `scripts/gateway_client.py` | AMD client、環境變數、重試、主控台編碼 | 在家重寫為通用服務層 |
| `scripts/index_writer.py` | SHA-256、FAISS、JSONL/manifest、碰撞檢查、暫存與回滾 | 最高 |
| `scripts/retrieval.py` | BM25、RRF、LLM rerank | 中高 |
| `scripts/render_pages.py` | 指定頁渲染為 PNG，補查圖表資訊 | 中 |

原版是 `scripts/` 中的平面模組，互相以裸模組名稱 import；`tests/conftest.py` 把 `scripts/` 加入 `sys.path`。在個人 repo 可改為正式 Python package，但需要同步改 imports 與 CLI entry points。

## 2. 系統契約與資料流

### 2.1 匯入流程

```text
PDF 路徑
  → pypdf 逐頁抽字、Symbol PUA 修正、重複行清理
  → 品質門檻（不合格就停，不花模型呼叫）
  → dry-run：只估塊數與呼叫數，到此結束
  → 計算原 PDF SHA-256，尋找可用續跑快取
  → 文件 review：標題／種類／摘要／章節起始頁
  → 決定 store 名稱，檢查輸出檔名碰撞
  → 依章節和字元預算切塊
  → 每塊產生簡述，定期 checkpoint
  → 將「簡述 + 原文」批次 embedding，逐批 checkpoint
  → 建 FAISS IndexFlatIP，寫三個輸出檔
  → 成功後清掉該次 partial cache
```

品質門檻在任何 API 呼叫之前執行。`--dry-run` 也不會建立 client、不做文件 review、不寫檔。正常流程的 LLM review 只呼叫一次；摘要預設每塊一次；embedding 以 32 筆為批。實際 LLM/API 消耗仍受 provider、重試及中斷續跑影響。

### 2.2 查詢流程

```text
問題 + store 名稱／目錄
  → 載入 .faiss、_meta.jsonl、可選的 _manifest.json
  → 用建立索引時的 embedding 模型向量化問題
  → FAISS 向量候選（預設 20）
  → BM25 文字候選（預設 20）
  → RRF 融合、去重
  → 可選 LLM rerank，保留預設前 5 筆
  → 將原文片段與頁碼交給回答模型
  → 輸出答案、檢索方法、來源頁碼
```

`--no-hybrid` 直接採向量前 `top-k`，也就不會呼叫 rerank；`--no-rerank` 保留 BM25+向量+RRF，但跳過 LLM 重排。查詢若缺 manifest，原版以第一筆 record 的 `source` 作標題備援。

### 2.3 多文件

每份 PDF 各自匯入，建立自己的三個檔案。跨文件問題的做法是對每個 store 各查一次，再於回答層整合，保持每句話能回溯至正確文件與頁碼。程式沒有跨 store 一次檢索或索引合併功能。

## 3. 原版 CLI 與使用者互動

### 3.1 匯入

```text
python scripts/ingest.py PDF_PATH --dry-run
python scripts/ingest.py PDF_PATH --out-dir OUTPUT_DIR --store-name STORE
python scripts/ingest.py PDF_PATH --out-dir OUTPUT_DIR --store-name STORE --resume
```

| 參數 | 原版預設 | 效果／限制 |
|---|---:|---|
| `pdf_path` | 必填 | 來源 PDF |
| `--out-dir` | `~/Documents/VectorDB4RAG` | 輸出與續跑 cache 的目錄；代理流程先詢問使用者存放處 |
| `--store-name` | 檔名 stem + LLM 偵測標題 | 清理為 ASCII `A-Z a-z 0-9 . _ -`，自動名稱最多 80 字元 |
| `--no-summaries` | 關閉此旗標，即有摘要 | 跳過逐塊 LLM 摘要，embedding 只用原文 |
| `--max-chunk-chars` | 1500 | 必須正整數 |
| `--chunk-overlap` | 150 | 必須 `0 <= overlap < max_chars` |
| `--dry-run` | false | 抽字、品質檢查、估算；不呼叫模型、不寫檔 |
| `--force` | false | 已有目標輸出時允許取代 |
| `--resume` | false | 讀可用 partial cache；沒有匹配 cache 時仍視為新跑並檢查碰撞 |
| `--embedding-provider` | `amd-gateway` | 原版只有 `amd-gateway` 一個實作 |

自動 store 名稱是經過 review 後才知道，故未明示 `--store-name` 時，碰撞檢查會在那次 LLM 呼叫之後。非拉丁標題可能被清理成只剩 PDF 檔名；中文文件宜明示簡潔名稱。`--dry-run` 的塊數是沒有章節資訊的估值，並非保證。

### 3.2 查詢

```text
python scripts/query.py "QUESTION" --store-name STORE --store-dir OUTPUT_DIR
python scripts/query.py "QUESTION" --store-name STORE --store-dir OUTPUT_DIR --json
```

| 參數 | 原版預設 | 效果 |
|---|---:|---|
| `question` | 必填 | 問題 |
| `--store-name` | 必填 | 對應三檔的共同前綴 |
| `--store-dir` | `~/Documents/VectorDB4RAG` | store 目錄 |
| `--top-k` | 5 | 最後交給回答 LLM 的片段數 |
| `--candidate-k` | 20 | 向量、BM25 各取幾筆；rerank 最多取融合前 20 筆 |
| `--no-hybrid` | false | 只用向量檢索 |
| `--no-rerank` | false | 不做 LLM 重排 |
| `--json` | false | 輸出機器可讀 JSON |
| `--embedding-provider` | `amd-gateway` | 原版以 CLI 選擇，未自動從 manifest 實例化 |

`query.py` 的文字輸出列出文件標題、檢索方法、前幾筆片段的分數與頁碼、最後的答案。JSON 欄位為 `question`、`store_name`、`doc_title`、`retrieval_method`、`answer`、`sources`；每個 source 含 `score`、`pages`、`section_title`。`retrieval_method` 可能為 `vector`、`hybrid`、`hybrid+rerank`。

### 3.3 圖表補查

`render_pages.py PDF_PATH PAGE [PAGE ...] --dpi 150 --out-dir DIR` 用 PyMuPDF 把指定的 **1-based** 頁碼轉為 `pN.png`。預設輸出到新的暫存目錄。當回答指出文字片段缺少圖表數字，先從檢索結果找頁碼，再查看命中頁及其相鄰幾頁。它不會把影像重新寫回索引，也不是 OCR。

## 4. PDF 擷取與品質檢查的細節

### 4.1 `pdf_extract.py`

逐頁資料模型是 `Page(number, raw_text, text)`；頁碼從 1 起算。`raw_text` 是 pypdf 擷取後經 Symbol-font 對映的文字；`text` 是去掉重複行並清理空白行的文字。`PdfReader` 建立後若加密，先嘗試空密碼；失敗給明確的密碼保護錯誤。接著強制 `len(reader.pages)`，因為破損頁樹有時懶惰解析到存取 `.pages` 才報錯。單頁 `extract_text()` 出錯則記成空字串，讓整份文件繼續處理；零頁文件直接失敗。

Symbol-font 修正針對 `U+F020` 至 `U+F0FF` 的 Private Use Area 字元，以 Adobe Symbol 對映表轉回希臘字母、數學比較／運算符號，未列入對映表的字元保持原樣。這對規格書中 `≥`、`≤`、`±`、`Ω`、`μ` 很重要；但若實際字型不是 Adobe Symbol，對映可能產生錯字。在家重寫可先保留這條規則並加註已知風險，若要精確處理須先辨別 PDF 字型。

重複行移除：對每頁每一行 `strip()`，計算該行出現在多少**不同頁**；出現頁數達 `max(2, int(總頁數 × 0.3))` 就視為 boilerplate，從所有頁去除。這通常刪掉相同頁首頁尾；也可能刪除真正反覆出現的內容。對頁碼會變的 footer，因為文字不同，這個規則不一定能去掉。原碼清理後會合併三個以上連續換行，保留最多兩個。

供 review 用的 page index 是每頁的第一條非空行，格式概念為 `pN: first line`；不是整頁摘要。這讓超長文件仍能向 LLM 提供大致結構，但對「每頁第一行都只是頁眉」的 PDF，章節偵測品質會下降。

### 4.2 `quality_check.py`

`assess()` 以清理後 `Page.text` 計算頁數、總字元、平均字元／頁、「少於 20 字元的頁」數及比例。`enforce()` 在以下任一條件成立時拒絕匯入：零頁；平均每頁少於 40 字元；或少字頁比例**大於** 50%。這是 `OR`，不是兩條件都要符合。結果是有些混合 PDF 即使有數頁文字，只要多數頁沒有文字，也會被拒；少量純圖頁混入正常文字文件則可通過。失敗訊息建議先用外部 OCR 工具產生文字層再重試。原版自身沒有 OCR，也不解析圖片中的表格、圖說或曲線。

## 5. 文件 review、切塊與摘要

### 5.1 文件 review

資料模型：`DocumentReview(title, doc_type, doc_summary, sections)`；每個 `Section` 有 `title` 與 1-based `start_page`。模型收到 PDF 檔名、總頁數、前 10 頁的清理後文字（合計最多 12,000 字元），以及全文件的精簡逐頁 index。要求回傳標題、文件類型、1–2 句文件概述、主要章節起始頁的 JSON。

逐頁 index 每行文字上限會隨頁數調整：以 `60000 // page_count` 計算，夾在 15 到 100 字元之間；整份 index 設有 150,000 字元硬上限，真的截斷時輸出警告。設計目的在於數百頁文件仍盡量覆蓋末段，而不是固定總長度、默默砍掉附錄。

LLM 可能在 JSON 前後補文字或插入大括號。原版先嘗試直接解析，再從每個 `{` 候選位置找配對 `}`，計算巢狀深度並忽略字串內的大括號，直到找到可解析的 JSON object。`sections` 中非 object、空標題、無法轉整數或超出頁碼範圍者會跳過；有效章節按起始頁排序。重寫時可用後端的結構化輸出功能，但仍需驗證欄位型別與頁碼。

### 5.2 切塊

預設每塊最多約 1500 字元、重疊 150 字元；`MIN_PAGE_CHARS=300`。若有章節，先用各章節 `start_page` 和頁 1 作分界，各區段分開切，不讓塊跨章節；沒有章節則對全 PDF 走同一頁面累積路徑。逐頁加入待處理 buffer，累積內容長度達 300 字元即 flush；短頁會併入下一頁。flush 時以兩個換行串接頁面，再按字元預算分割。長文本優先在最後一個換行或句點空白處切，否則硬切；下一塊從前一塊末尾向前重疊 150 字元。每塊資料為 `Chunk(text, pages, section_title)`。

**頁碼精度問題：** 原版 flush 後若拆出多塊，這些塊都拿到該次 buffer 的完整 `pending_pages` 清單，即使某塊文字只來自其中一頁。對長頁拆分還好，但跨頁合併後頁碼可能較寬；回答的頁碼應回原 PDF 核對。在家重寫可在組合文字時保留每段來源 offset，依塊的字符範圍算真實頁碼。`overlap >= max_chars` 會使推進步幅近乎 1，造成大量重複塊；CLI 必須提前阻擋。

### 5.3 每塊簡述

預設逐塊呼叫聊天模型，讓模型看文件標題、類型、章節名及該塊文字前最多 3000 字元，生成 1–2 句「此段在文件中的位置與用途」的簡述。簡述與原文分開存，embedding 輸入是 `summary + newline + text`；回答仍使用原始 `text`。`--no-summaries` 時無簡述，embedding 只用原文。原版 `SUMMARY_SCHEMA_VERSION=2`，這個版本值參與續跑相容性判斷；改 prompt 語意時應遞增。

## 6. Embedding、索引及資料格式

### 6.1 Embedding 抽象

原版 `EmbeddingProvider` 提供 `name`、`model_name`、`batch_size` 與抽象 `embed(texts) -> vectors`。共用的 `embed_by_id({chunk_id: text}, on_batch_done)` 按 id 排序並批次呼叫 `embed`，每批結束立刻回報 `{id: vector}` 以便 checkpoint。預設 batch size 32。只有 `AmdGatewayEmbedding`，模型是 `bge-m3`；透過 OpenAI 相容 embeddings endpoint 呼叫。

在家重寫時，介面至少驗證：回傳向量數等於輸入文字數；每個向量同維度、有限浮點數、非全零；回傳順序對應輸入順序。原版 `zip` 不會檢查長度，若服務少回向量，最後才以 missing ids 報錯。更換 provider 或模型後，應在 manifest 記下 provider、模型、維度與必要前處理設定；查詢時由 manifest 選擇並驗證，不能只信 CLI 預設值。

### 6.2 三個輸出檔

`<store>.faiss`：把矩陣轉成 `float32`，逐列 L2 normalize，建 `faiss.IndexFlatIP`，因此內積等於 cosine 相似度。`<store>_meta.jsonl`：每行一塊，**行號必須等於 FAISS 向量列號**。`<store>_manifest.json`：文件層級資訊。查詢是用 FAISS 回傳的位置直接索引 records，不另外依 `id` 搜索；索引與 JSONL 若錯位會拿錯原文。

每筆 JSONL 的原版 schema：

```json
{"id":0,"text":"原文段落","pages":[12,13],"source":"example.pdf","summary":"脈絡簡述或 null","section_title":"章節名或 null"}
```

manifest 的主要欄位：`schema_version=1`、`source_pdf`、`source_pdf_sha256`、`doc_title`、`doc_type`、`doc_summary`、`sections`、`embedding_model`、`embedding_provider`、`chat_model`、`chunk_count`、`chunk_settings`（`max_chars`、`overlap`、`summary_schema_version`）、`summaries_enabled`、`tool_version`、UTC `created_at`。原版 `source_pdf` 只存檔名，不存完整來源路徑；若要跨機器可重現，宜另設由使用者指定的來源識別欄位。

### 6.3 寫入與碰撞

若任何目標三檔已存在且無 `--force`，報錯停止。輸出先寫到同目錄的 `.tmp` 路徑；內容都寫完後逐檔取代。原有檔案先改名 `.bak`，若中途 rename 失敗，刪除已換上的新檔、恢復備份；全部成功後刪除備份。這提供一般失敗情況下的三檔一致性，但三次 rename **不是單一原子交易**；遇到程序被強制終止、斷電或回滾本身失敗，仍須有復原策略。個人版可考慮寫入新版本目錄，完成驗證後以單一指標檔切換版本。

## 7. 續跑 cache：這份設計最值得保留的經驗

暫存位置：`<out-dir>/.pdf_to_rag_cache/<store>.partial.json`。內容包括來源 PDF SHA-256、以字串化 chunk id 為 key 的 `summaries` 與 `embeddings`、整份 `review`、`chunk_settings`。摘要每新增到第 10 筆或最後一筆會 checkpoint；摘要階段完畢再無條件 checkpoint；embedding 每批完成 checkpoint；中斷或例外也 checkpoint。成功寫入三輸出檔才清除 cache。

`--resume` 若有明示 store 名，直接找該名的 cache；若無，掃描同一輸出目錄下的 `.partial.json`，找來源 SHA-256 相同者，取修改時間最新的一份，並沿用其**原來的 store 名及 review**。這避開重新呼叫 LLM 後標題文字不同，導致自動檔名漂移。來源 hash 不合則不沿用。chunk 參數或摘要版本不合則丟棄整份 cache，避免相同 chunk id 指向不同文字或不同摘要語意。

三個不能漏的條件：

1. 碰撞判斷依「**真的找到且可用 cache**」執行，不依使用者有沒有打 `--resume`。找不到 cache 的續跑其實是新跑，不能默默蓋舊 store。
2. 任何改變 chunk id 對應內容或 embedding 輸入的設定都要進 fingerprint。原版檢查字元大小、重疊、摘要 schema；在家版還應加入 embedding provider/model、正規化及 prompt/config 版本。
3. 所有 checkpoint 位置必須寫相同的 `review` 與設定；原版用單一 closure 避免失敗路徑漏欄位。個人版應再把 cache 寫入改為 temp+replace，避免寫到一半使 JSON 壞掉。

注意：目前原版允許沒有 `chunk_settings` 的舊 cache（以 `None` 通過），且 cache fingerprint 沒有 embedding provider/model。若換了 embedding 後仍沿用同一個中斷 cache，可能混用不同模型的向量；這是重寫時應修的相容性風險。`--no-summaries` 與前次摘要設定切換，也應明確當成不同輸入配方，避免沿用不合適的摘要或向量。

## 8. 查詢演算法詳解

### 8.1 向量檢索

問題用同一 embedding 服務轉為 `float32` 向量，L2 normalize，對 FAISS `IndexFlatIP.search()` 取前 `candidate-k`。FAISS 回傳 `-1` 的空位置略過。score 是內積／cosine（因兩側都已正規化）。若改用其他相似度或量化索引，分數含義須重寫。

### 8.2 BM25 與 RRF

BM25 在每次查詢時從 JSONL record 的 `text` 重建 `BM25Okapi`；斷詞是小寫後 regex `\w+`。它對英文關鍵字、型號及數字較有用；繁中、簡中、日文沒有自然空白分詞，效果較弱。原版即使所有 BM25 分數為 0，也仍取前 `candidate-k` 筆進融合，可能使資料順序影響 RRF。個人版可先量測，再決定是否排除零分或加入合適的中文斷詞。

RRF 計算每份排名清單的 `1 / (60 + rank)`，rank 從 1 算起，同一筆出現在兩份清單就相加；不要直接加 cosine 與 BM25 原始分數。原版以同一份 `records` 陣列中 dict 物件的 Python `id(record)` 去重，因兩路搜尋都回傳對相同 dict 的引用。若重寫成不同進程、複製 dict 或序列化結果，必須改用明確穩定的 chunk ID 去重。

### 8.3 LLM 重排與回答

融合後最多取前 `candidate-k` 筆給 reranker。每筆標一個候選序號並送原文前 1000 字元，要求模型回 `{"ranked_ids":[...]}`。解析失敗或 API 例外時回退到原 RRF 次序；回傳重複、越界、缺漏 ID 則略過無效項，依原 RRF 順序補滿 `top-k`。因此重排故障不應使查詢完全失效。

最後回答 prompt 帶文件標題、每塊頁碼與章節、完整原文，要求**只根據提供的片段**作答、標註頁碼；若片段不足，應明說。回答之後若發現圖表細節不足，須回原 PDF 渲染相關頁確認。原版輸出的 `sources.score` 在混合路徑是 RRF 分數，不是 reranker 的新分數；重排只改次序，解讀分數時要注意。

## 9. 把 AMD Gateway 抽掉的實作設計

原版 `gateway_client.py` 建 OpenAI 相容 client，讀 `LLM_GATEWAY_KEY`，設定 AMD 的 `base_url` 與特殊 subscription header。它也負責重試和 Windows UTF-8 主控台設定。模型常數散落在 `llm_review.py`、`summarize.py`、`query.py`、`retrieval.py`；embedding 模型在 `embedding.py`。在家重寫可使用下列分層，不讓業務邏輯 import 特定廠商 client：

```text
ChatService:
  review_document(prompt) -> validated DocumentReview
  summarize_chunk(prompt) -> str
  rerank(prompt) -> list[int]
  answer(prompt) -> str

EmbeddingService:
  model_id / provider_id / dimension
  embed(list[str]) -> list[list[float]]

Infrastructure:
  retry policy、timeout、key/config 載入、日誌、主控台編碼
```

選用同一服務或分開服務都可以；依你的個人電腦條件決定是本機推論、雲端 API 或兩者混合。這份文件不預設特定供應商、模型名稱、價錢或可用性。若採 OpenAI 相容 API，通用設定可包括 `CHAT_BASE_URL`、`CHAT_API_KEY`、`CHAT_MODEL`、`EMBED_BASE_URL`、`EMBED_API_KEY`、`EMBED_MODEL`，但實際環境變數名稱由你新 repo 自訂。密鑰只放使用者環境或不納入 Git 的 `.env`；不要沿用 AMD 的 URL、header、金鑰或內部模型名稱。

重試策略可沿用原版概念：429、連線錯誤、逾時、伺服器 5xx 最多 6 次，從 5 秒指數退避、上限 90 秒，另加至多 25% jitter；400/401/403 及資料解析錯誤應直接報錯。摘要、review、embedding 失敗會留下 checkpoint；rerank 失敗則回退。回答模型失敗在原版未包一層友善錯誤訊息，個人版可補上。

**相容性不可省略：** 舊 store 是由 `bge-m3` 的向量空間建立，換一個 embedding 模型後，即使維度恰巧一樣，也不能拿新模型的 query vector 搜舊索引。請用新模型重新匯入 PDF，並以 manifest 的模型識別驗證查詢配置。若重寫改變摘要 prompt、切塊、文字清理或向量正規化，也應使 store 版本明確變更。

## 10. 測試與重建順序

原資料夾 `requirements.txt` 列出 `openai>=1.0.0`、`python-dotenv>=1.0.0`、`pypdf>=4.0.0`、`faiss-cpu>=1.7.4`、`numpy>=1.24.0`、`pymupdf>=1.24.0`、`rank_bm25>=0.2.2`；文件宣稱 Python 3.10+。若移除 AMD 或更換 SDK，依新 provider 調整依賴。原版 `pytest.ini` 只指定 `testpaths = tests`。我在原資料夾執行 `python -m pytest -q`，結果 **87 passed in 4.05s**；README/AGENTS 仍寫 86，已與目前測試數不同。這些測試主要是純邏輯與 mocked API，不代表在家後端或真實 PDF 已驗證。

重寫的實用順序：

1. 定義 `Page`、`Section`、`DocumentReview`、`Chunk`、meta record 與 manifest 的 schema；先固定頁碼、ID、排序與模型識別規則。
2. 做 PDF 擷取、文字品質門檻、無模型的 fallback 切塊；用自己建立的極小文字 PDF 檢查整條本地資料流。
3. 做 `EmbeddingService`、批次向量、FAISS 寫入與讀取；先用假 embedding 服務驗證 row 對齊、維度和碰撞。
4. 加文件 review 與摘要；即使 LLM JSON 不符合 schema，仍要有明確錯誤或可控制的 fallback。
5. 加 cache 與 `--resume`；優先測來源 hash、設定 fingerprint、找不到 cache 的碰撞保護、每批 checkpoint、失敗後恢復。
6. 做純向量查詢，再加 BM25／RRF／rerank／回答。每階段都用同一小文件核對檢索與頁碼。
7. 最後加 `render_pages.py` 及 SKILL/README/EXAMPLES 文件，把使用者互動（先確認輸出位置、長文件先 dry-run、回報三檔路徑）寫回新 skill。

原測試的重點分布：`test_pdf_extract.py` 測字元對映／重複行／page index；`test_quality_check.py` 測空頁比例；`test_llm_review.py` 測 JSON 解析、長文件 index、壞章節；`test_chunking.py` 測切點、overlap 與章節；`test_embedding.py` 測批次及 callback；`test_gateway_client.py` 測重試與 env 搜尋；`test_index_writer.py` 測快取、碰撞、三檔寫入及 rename 中途失敗回滾；`test_retrieval.py` 測 BM25、RRF 物件去重、rerank 失敗回退與缺漏補齊。原版沒有把真 PDF 擷取、完整 ingest/query CLI、不同 embedding 模型或真實 API 放入這套可攜 pytest。

## 11. 原版已知限制與在家版優先修正清單

| 事項 | 原版實際行為 | 重寫建議 |
|---|---|---|
| 掃描 PDF | 無 OCR，品質門檻拒絕 | 保持清楚拒絕，或日後獨立加 OCR；不要默默索引空文字 |
| 圖表／影像文字 | 不進索引，查詢後人工 render | 保留按需 render；量測需求後才考慮圖說或多模態 |
| 頁碼 | 一次 flush 拆多塊時共享所有待處理頁碼 | 追蹤字元 offset 與真實頁碼範圍 |
| BM25 中文 | `\w+` 無中文分詞；零分仍可能入候選 | 依語言評估斷詞與零分過濾 |
| store 查詢模型 | CLI 預設 provider，manifest 只用於標題 | 由 manifest 決定並驗證 embedding 模型／維度 |
| 續跑模型切換 | cache fingerprint 未記 provider/model | 把模型及所有影響 embedding 的配方納入 fingerprint |
| cache 檔 | 直接 `write_text`，壞 JSON 會被忽略 | temp+replace，並留下可辨識的損壞診斷 |
| 輸出三檔 | temp+backup+回滾，非真正單一交易 | 加版本目錄或發布指標；載入時驗證 `ntotal == records` |
| BM25 索引 | 每次 query 重新建立 | 大 store 可儲存可重建的詞項索引或加快取 |
| 參數檢查 | chunk size/overlap 已檢；query 的 `top-k`/`candidate-k` 未明確檢 | 加正整數與範圍限制 |
| 語言／字型 | Symbol 對映假設 Adobe Symbol | 保留原字及轉換紀錄，必要時查字型 |
| 安全與路徑 | `.env` 搜尋會掃祖先及其直接子目錄 | 在家版用單一明確設定位置或環境變數，避免誤讀別的專案金鑰 |

## 12. 完成條件

在家版達到下列可觀測結果，就已重建原 skill 的核心能力：

- 對有文字層的 PDF，dry-run 不呼叫模型、不寫檔，能估頁數／塊數／呼叫數；掃描 PDF 給清楚錯誤。
- 正常匯入產生三檔，meta 行數與 FAISS `ntotal` 完全一致，manifest 記下來源 hash、模型、設定與建立時間。
- 中斷摘要或 embedding 後，同參數 `--resume` 不重做完成的項目；來源、模型或切塊配方改變時不沿用錯誤 cache。
- 重跑遇到既有 store 時不會因誤打 `--resume` 而覆蓋；`--force` 取代失敗時可恢復先前完整版本。
- 問題能以相同 embedding 模型搜尋到原文，混合檢索與 rerank 可開關；rerank 失敗仍可回答。
- 答案只用提供的片段並給頁碼；資料在圖表像素中時可渲染指定頁供核對。
- 兩份 PDF 維持各自索引，可分別查詢再帶著各自頁碼比較。

這份規格故意以可重新實作的介面、資料契約、演算法與錯誤案例呈現；原 skill 中的 AMD 專屬接入與測試文件本身都不需要帶回個人 repo。
