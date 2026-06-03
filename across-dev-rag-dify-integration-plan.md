# Across-Dev RAG 短期方案：Dify Retrieve API + across RBAC

> **產出日期**：2026-05-31
> **專案路徑**：`/Users/littlesmart/Documents/across-各版本/across-dev`
> **定位**：短期快速落地方案，保留向量庫彈性供長期評估
> **三方驗證**：Claude Opus 4.6 + Gemini 2.5 Pro + OpenAI Codex (GPT-5.5)
>
> **⚠️ 本文件提到的所有地端模型名稱（LLM、Embedding、Reranker）都是候選清單，尚未決定。** 每個模型都需要驗證能否在地端正常運行、授權條款是否允許商用、中英文實際品質，全部待測試（見第八章、第十二章）。

---

## 一、目標

在 across 平台加入 RAG 知識庫功能：

- MIS 可透過管理頁面新增、刪除、修改、查詢知識庫，上傳文件（支援 .md、.docx、.pdf、.txt，中文和英文），指定知識庫綁定到哪些角色。文件內的圖片不會被收錄、掃描件 PDF 不支援、表格資料會丟失（結構會跑掉）
- 被綁定的角色在 AIW Copilot 對話視窗中可基於該角色對應的知識庫內容進行問答，需支援中文和英文
- 不同角色只能查詢被 MIS 授權的知識庫（RBAC）
- 現階段各部門角色不能自行新刪修查自己部門的知識庫，此功能列入中期規劃

---

## 二、部署限制


| 限制               | 說明                                |
| ---------------- | --------------------------------- |
| 地端部署（Air-gapped） | 客戶主機不能上網，所有模型必須本地推論               |
| 多租戶 VM           | 一台物理主機跑多個 VM，每個 VM 一套 across，資源互搶 |
| LLM 推論引擎         | **vLLM**，OpenAI-compatible API    |
| 語言               | 文件和回答需支援中文和英文                     |


### 客戶主機硬體


| 方案         | CPU                         | RAM        | GPU                             | VRAM       |
| ---------- | --------------------------- | ---------- | ------------------------------- | ---------- |
| BOM 1 (×3) | Intel Xeon 6 24C            | 256GB DDR5 | RTX PRO 6000 Server Edition ×1  | 96GB GDDR7 |
| BOM 2      | Intel Xeon 6 48C (2P)       | 256GB DDR5 | RTX PRO 6000 Blackwell Max-Q ×1 | 96GB       |
| BOM 3      | Threadripper PRO 7975WX 32C | 512GB DDR5 | RTX PRO 6000 Blackwell Max-Q ×4 | 96GB ×4    |


---

## 三、時程評估

> **⚠️ 以下時程為預估，實際開發和測試過程中如遇到突發狀況（模型在 vLLM 上跑不起來、Dify container 省略後功能異常、共用 postgres/redis 衝突、rerank API 格式不相容等），時間會拉長。每個階段都保留彈性空間。**

### 階段 1：RAG 服務跑通

不做模型選型比較，每種模型先選一個能跑的，目標是讓整條 RAG pipeline 在 across 上跑通。

| 工作 | 預估時間 |
|------|---------|
| **開發（AI 協助）** | |
| B. 資料庫 schema + migration | 2-3 小時 |
| C. 知識庫管理 API（串接 Dify API + 角色綁定） | 6-8 小時 |
| D. kb-rag Worker（tool 定義、撈 prompt、Dify Retrieve client、RBAC 查詢、並行檢索、rerank 改寫、agent.md、整合除錯） | 11-19 小時 |
| A. Dify 整合進 across docker-compose + 設定 | 6-10 小時 |
| **開發小計** | **約 25-40 小時（4-5 個工作天）** |
| **測試（只測能不能跑通）** | |
| 共用 postgres/redis 能不能跑 | 2-4 小時 |
| Dify 最小 container 驗證 | 4-8 小時 |
| 端到端跑通 | 2-4 小時 |
| RBAC 正確性 | 2-4 小時 |
| 上傳到可查詢 | 2-4 小時 |
| **測試小計** | **約 12-24 小時（2-3 個工作天）** |
| **階段 1 合計** | **約 37-64 小時（5-8 個工作天）** |

階段 1 不做的事：模型選型比較、模型資源佔用量測、top_k 策略調優、rerank chunks 數量調優、多次 LLM 問題觀察、上傳量控制。

> **階段 1 的時程是在一切順利的情況下估算的。** 可能拉長時間的因素包括：選的模型在 vLLM 上跑不起來需要換一個、Dify 省略 container 後功能異常需要加回來除錯、共用 postgres/redis 出現衝突需要排查、rerank API 格式與現有程式碼不相容需要調整、across 的 Worker + tool 架構串接時出現預期外的問題等。

### 階段 2：完整短期項目

在階段 1 跑通的基礎上，補上模型選型、資源量測、品質調優等所有第十二章的測試項目。

| 工作 | 預估時間 |
|------|---------|
| 模型選型測試（LLM + Embedding + Reranker 候選比較） | 20-40 小時（含模型下載、跑 benchmark、比較結果） |
| 模型資源佔用量測（SSD、VRAM、GPU-Util、DRAM） | 含在模型選型內 |
| top_k 策略調優（動態 vs 固定） | 4-8 小時 |
| rerank chunks 數量調優 | 含在 top_k 調優內 |
| 多次 LLM 問題觀察 | 4-8 小時 |
| 上傳量控制測試 | 4-8 小時 |
| 整合測試 + 修 bug | 8-16 小時 |
| **階段 2 小計** | **約 40-80 小時（5-10 個工作天）** |
| **階段 1 + 2 合計** | **約 77-144 小時（10-18 個工作天）** |

> **彈性說明**：模型選型是最大的不確定因素 — 如果候選模型多、每個都要測資源佔用和品質，時間會往上限靠。如果第一個測的模型就夠用，時間會往下限靠。測試過程中發現的 bug 修復時間也無法預估。


---

## 四、方案選型過程

### 評估過的 RAG 平台


| 平台          | 中資  | 公司 / 所在地                                   | 淘汰原因                                                                                                                                 |
| ----------- | --- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| RAGFlow     | 是   | InfiniFlow 英飛流 / 上海                        | 向量庫只支援 ES 和 Infinity（僅 2 種）。RAGFlow 使用 ES 8.14.3，不支援 ES 9.x，無法共用 across 現有的 ES 9.3（跑 log 已有壓力），需額外起一個 ES 8.x container，兩個 ES 同時跑資源太重 |
| FastGPT     | 是   | 珠海環界雲計算（Labring）/ 珠海，阿里雲獨家投資               | 無父子切片、向量庫只支援 pgvector/Milvus                                                                                                         |
| MaxKB       | 是   | FIT2CLOUD 飛致雲 / 杭州                         | 沒有明確的 Retrieve API、GPL-3.0                                                                                                           |
| Open WebUI  | 否   | Open WebUI Inc. / 加拿大，創辦人 Timothy Baek（韓裔） | 沒有 Retrieve API                                                                                                                      |
| Kotaemon    | 否   | Cinnamon AI / 東京，創辦人平野未來（日本）               | 沒有 API                                                                                                                               |
| AnythingLLM | 否   | Mintplex Labs / 美國加州，YC S22                | 沒有 hybrid search、中文支援差                                                                                                               |
| Quivr       | 否   | QuivrHQ / 法國巴黎，YC W24                      | 部署要 10+ container、API 不完整                                                                                                            |
| Langflow    | 否   | 原巴西 Logspace，被美國 DataStax 收購               | 工作流建構器，不是知識庫系統                                                                                                                       |
| LlamaIndex  | 否   | LlamaIndex / 美國舊金山                         | 程式庫不是平台，2 週內無法做完                                                                                                                     |


> **備註**：本方案選用的 Dify 也是中資。LangGenius 創辦人張路宇（中國安徽），原始公司註冊於蘇州（苏州语灵人工智能科技有限公司），後於美國德拉瓦州設立 US entity 進行全球化擴展，投資方包含阿里雲。在評估過的 10 個平台中，4 個為中資（Dify、RAGFlow、FastGPT、MaxKB）。

### 為什麼選 Dify

1. **向量庫彈性**：Dify 支援 30+ 種向量庫，短期沿用 Weaviate，中期評估選型時不被鎖死
2. **中文支援**：內建 jieba 分詞
3. **父子切片**：有（v0.15.0+）
4. **混合搜尋**：有（向量 + 全文）
5. **Rerank**：支援外接（Cohere、Jina、bge-reranker 等）

### 為什麼不用 Chatflow 而用 Retrieve API


| 問題              | 說明                                                           |
| --------------- | ------------------------------------------------------------ |
| RBAC            | across 角色無上限，Chatflow 的 dataset_ids 是固定的，無法 per-request 動態切換 |
| 多 App 方案        | 角色無上限 = App 數量無上限，不可行                                        |
| Dify Enterprise | KB 權限（部分成員）只在 UI 生效，API 層不生效（三方驗證確認）                         |
| 結論              | RBAC 只能由 across 控制，用 Retrieve API 最直接                        |


#### 什麼是 Retrieve API？

Dify 提供兩種呼叫知識庫的方式：

1. **Chat API**（`/v1/chat-messages`）：把問題丟給 Dify 的 Chatflow/Chatbot App，Dify 內部自動完成「知識檢索 → Rerank → 組 context → 呼叫 LLM → 回答」整條流程，回傳的是 LLM 生成的完整回答。across 只是轉發，無法控制中間過程。
2. **Retrieve API**（`POST /v1/datasets/{dataset_id}/retrieve`）：只做「知識檢索」這一步，回傳的是相關的**原始文件片段（chunks）**和相似度分數，不呼叫 LLM、不生成回答。across 拿到 chunks 後自己決定怎麼處理（合併、rerank、組 prompt、呼叫 LLM）。

本方案選擇 Retrieve API，因為 across 需要在「檢索」和「回答」之間插入 RBAC 控制邏輯（決定查哪些 KB），這在 Chat API 的封閉流程中做不到。

> **三方驗證結論**（Claude + Gemini + Codex 一致確認）：
>
> Dify 原生 App API / Knowledge API 沒有提供「依你們產品使用者身份，自動限制可查知識庫」的 runtime ACL。Enterprise 的 Web App 權限也不會套用到 API。要做到這件事，必須由 across 後端控 API Key / App 路由 / metadata filter / external retrieval service。
>
> **「一個角色一個 App」為什麼不可行：**
>
> across 產品角色無上限，不管是 Community 還是 Enterprise，用「一個角色 / 部門一個 Dify App」都會有爆炸問題：
>
>
> | 版本                          | 能不能建很多 App + 很多 API Key               | 能不能解決角色無上限爆炸              |
> | --------------------------- | ------------------------------------- | ------------------------- |
> | Dify Community self-hosted  | 理論上可以，受機器、DB、維運成本限制                   | 不能根本解決                    |
> | Dify Enterprise self-hosted | 可以，且有較完整的團隊、群組、Web App access control | 還是不能根本解決 API 查詢時的產品角色權限問題 |
> | Dify Cloud paid plans       | 有方案資源限制，不適合地端情境                       | 也不是這題的解法                  |
>
>
> Dify 官方文件寫 Community / Enterprise self-hosted 的 workspace members 是 unlimited，但這指的是 Dify 工作區成員數，不是「across 產品角色數」或「App API 依產品使用者自動控知識庫」。
>
> **真正關鍵**：Dify Web App 權限不影響 API access；API access 由 API Key 控制。官方文件明確寫 "changing web app permissions doesn't affect existing API functionality"，API 文件也寫所有 API request 都是用 `Authorization: Bearer {API_KEY}`。

---

## 四、架構

> **名詞說明**：本文件中的 **KB**（Knowledge Base）指的是 Dify 裡的一個「知識庫」（Dify 稱為 dataset）。一個 KB 可以包含多份文件，每份文件被切片後儲存為多個 chunks。across 透過 RBAC 控制不同角色可以存取哪些 KB，例如資安人員可讀 KB-A（資安規範）和 KB-B（事件報告），行銷人員只能讀 KB-C（行銷資料）。

### 4.1 職責分工


| 步驟                               | 誰做                                                              | 說明                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| -------------------------------- | --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 建知識庫、上傳文件、切片、embedding           | **Dify**                                                        | Dify Worker 處理                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| 向量檢索（Retrieve）                   | **Dify**（per KB）+ **across** Worker（新寫）                         | Dify Retrieve API 一次只能查一個 dataset。多 KB 並行呼叫（Promise.all）再合併的邏輯由 across Worker 新寫                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| 知識庫管理（新刪修查 KB、設定角色權限）            | **across** api-admin（新寫）                                        | MIS 專用功能（短期由 MIS 統一管理，不開放各部門自管）                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| RBAC + 檢索 + 合併 + Rerank（透過 tool） | **across** kb-rag Worker 的 `kb-rag-retrieve` tool execute 內（新寫） | Worker LLM 呼叫 tool，tool 的 execute 函式裡用程式碼依序執行（不需改 `execute.ts`，用現有 Worker + tool 架構）：(1) 查 RBAC：查 DB 取得該使用者角色可讀的 KB 列表。(2) 並行檢索：Dify 的 Retrieve API（`POST /v1/datasets/{dataset_id}/retrieve`）一次只能查一個 dataset，不支援在一個 request 帶多個 dataset_ids。但 Dify Chatflow 內部的 `multiple_retrieve` 其實做得到多 KB 並行檢索 + 合併 + rerank，只是沒有開放在公開 API 上。因此 tool 內部用 Promise.all 對每個允許的 KB 並行呼叫 Retrieve API，動態 top_k（總 chunk 預算 / KB 數），合併成一個陣列。自建版 Dify 無 API rate limit，併發不受限。(3) Rerank：全部 KB 的 chunks **合併後一次 rerank**（不是個別 KB 分開 rerank），用原始 prompt + 全部 chunks 呼叫 LiteLLM → vLLM bge-reranker → 取 top N。此做法與 Dify Chatflow 內部的 multiple_retrieve 邏輯一致。(4) 回傳 top-K chunks 給 Worker LLM |
| 生成回答                             | **across** kb-rag Worker LLM（機制現有，能力待驗證）                        | Worker LLM 看到 tool 回傳的 top-K chunks 後生成回答。Worker 用自己的 LLM model，可獨立設定，不影響 Commander 和其他 Worker 的 model（見第八章）。整個 Worker 執行共 2 步（Step 1 呼叫 tool + Step 2 生成回答）                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| 前端 — 知識庫管理頁面                     | **across** web（新寫）                                              | MIS 專用頁面：建立/刪除知識庫、上傳文件、指定哪個知識庫綁在哪個角色上。短期不做各部門自行管理知識庫的功能                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| 前端 — 問答                          | **across** web（現有）                                              | 使用者在現有 Commander 對話視窗提問即可                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |


### 4.2 架構圖

```
┌─────────────────────────────────────────────────────────────────┐
│  每個客戶 VM                                                     │
│                                                                 │
│  ┌─────── across（現有）──────────────────────────────────────┐  │
│  │                                                           │  │
│  │  ┌────────┐   ┌──────────────────────────────────────┐    │  │
│  │  │  web   │──▶│ agent-server                         │    │  │
│  │  │  AIW   │   │                                      │    │  │
│  │  │Copilot │   │ Commander (現有 orchestrator)         │    │  │
│  │  │        │   │  └→ 派任務給 kb-rag Worker (新增)     │    │  │
│  │  └────────┘   │                                      │    │  │
│  │               │ kb-rag Worker (新增, 自己的 LLM model)│    │  │
│  │               │                                      │    │  │
│  │               │  Step 1: Worker LLM 呼叫 tool        │    │  │
│  │               │    tool execute 裡跑程式碼：          │    │  │
│  │               │      0. 從 session DB 撈原始 prompt   │    │  │
│  │               │      1. 查 RBAC → 角色可讀 [KB-A,B]  │    │  │
│  │               │      2. Promise.all 並行打 Dify API   │    │  │
│  │               │         (動態 top_k = 預算/KB數)     │    │  │
│  │               │      3. 全部 chunks 合併成一個陣列    │    │  │
│  │               │      4. 合併後一次 rerank → top N     │    │  │
│  │               │    → 回傳 top-K chunks 給 Worker LLM  │    │  │
│  │               │                                      │    │  │
│  │               │  Step 2: Worker LLM 看到 chunks      │    │  │
│  │               │    → 生成回答                         │    │  │
│  │               │    → 回傳答案給 Commander             │    │  │
│  │               │  Commander 轉發回答給 User            │    │  │
│  │               │  ⚠️ 地端LLM能否中英文回答待驗證(§8) │    │  │
│  │               └─────────────┬───────────────────────┘    │  │
│  │                             │                            │  │
│  │  ┌────────┐  ┌───────┐      │     ┌──────────────────┐   │  │
│  │  │postgres│  │ redis │      │     │ LiteLLM          │   │  │
│  │  │(across)│  │(across)│     │     │ → vLLM (rerank)  │   │  │
│  │  └────────┘  └───────┘      │     │ → vLLM (LLM)     │   │  │
│  │                             │    └──────────────────┘    │  │
│  └─────────────────────────────┼────────────────────────────┘  │
│                                │                               │
│  ┌─────── Dify（新增）──────────┼────────────────────────────┐  │
│  │                             ▼                            │  │
│  │  ┌──────────┐  ┌──────────────┐  ┌──────────────────┐    │  │
│  │  │ dify-api │  │ dify-worker  │  │ Weaviate         │    │  │
│  │  │ Retrieve │  │ 切片+embedding│  │ 向量儲存          │    │  │
│  │  │ API      │  │              │  │ （沿用現有）     │    │  │
│  │  └──────────┘  └──────────────┘  └──────────────────┘    │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌─────── vLLM（新增，GPU）─────────────────────────────────┐   │
│  │  ├── LLM (待定)              ~?GB VRAM                   │   │
│  │  ├── bge-m3 (embedding)      ~2GB VRAM                   │   │
│  │  └── bge-reranker-v2-m3      ~2GB VRAM                   │   │
│  │  OpenAI-compatible API                                    │   │
│  └───────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**架構說明：**

整個架構分為三層，全部跑在同一個客戶 VM 內：

1. **across（現有 + 新增 Worker）**：核心控制層。使用者透過 AIW Copilot 頁面操作，所有請求進入 agent-server 的 Commander。Commander 是現有的 orchestrator，透過 Redis message bus 派任務給 Worker。本次新增一個 `kb-rag` Worker agent，Commander 判斷使用者問題需要查知識庫時派任務給它。Worker 內部負責：查 RBAC 決定可讀的 KB → 並行呼叫 Dify Retrieve API → 合併 chunks → 呼叫 rerank 精排 → 組 top-K context → **用 Worker 自己的 LLM model 生成回答**。Worker 的 model 可獨立設定（`agent.model` 欄位），不影響 Commander 和其他 Worker。此外，MIS 有專用的知識庫管理頁面（透過 api-admin），可建立/刪除知識庫、上傳文件、指定知識庫綁定到哪些角色。短期由 MIS 統一管理，不開放各部門自行新刪修查知識庫。postgres 和 redis 是 across 現有的基礎設施，同時共用給 Dify（獨立 database / db number，互不干擾）。**共用是否會互相影響需要測試驗證**（見 4.5 共用注意事項）。LiteLLM 是 across 現有的模型路由層，本次需新增 vLLM 的 rerank 和本地 LLM endpoint 設定。
2. **Dify（新增服務）**：知識庫引擎層。dify-api 提供 Retrieve API 供 across 呼叫，每次只查單一知識庫、回傳原始 chunks（不呼叫 LLM）。dify-worker 負責文件上傳後的非同步處理（解析 → 切片 → 呼叫 vLLM 算 embedding → 寫入 Weaviate）。Weaviate 是沿用 Dify 現有的向量儲存，中期再評估是否更換。
3. **vLLM（新增，GPU）**：模型推論層。跑在 GPU 上，提供 OpenAI-compatible API。包含三個模型：LLM（待評估選型）負責生成回答、bge-m3 負責 embedding（文件向量化和查詢向量化）、bge-reranker-v2-m3 負責重排序（模型皆為候選，待評估確認）。

**資料流向**：使用者在 Commander 對話視窗提問 → Commander 判斷需要查知識庫 → 透過 Redis bus 派任務給 kb-rag Worker → Worker 內部查 RBAC → 對允許的每個 KB 各打一次 Dify Retrieve API → 合併所有 chunks → 經 LiteLLM 呼叫 vLLM reranker 精排 → 組 top-K context → Worker 用自己的 LLM model 生成回答 → 透過 Redis bus 回傳給 Commander → Commander 轉發回答給前端。

### 4.3 Dify 最小部署

目前 Dify 本地完整部署跑了 11 個 container。目標是只保留知識庫（建立、上傳、切片、embedding、Retrieve API）所需的最小集合，其餘共用 across 現有服務或省略。

**確定必要的 container：**


| Container     | 用途                                     | 必要性 |
| ------------- | -------------------------------------- | --- |
| `dify-api`    | Knowledge Base API + Retrieve endpoint | 必要  |
| `dify-worker` | 非同步處理文件（切片、embedding）                  | 必要  |
| `weaviate`    | 向量儲存（沿用 Dify 現有）                       | 必要  |


**預計共用 across 現有服務（需驗證 Dify 能正常指向外部服務）：**


| Container     | 共用方式                                            | 待驗證事項                                      |
| ------------- | ----------------------------------------------- | ------------------------------------------ |
| `db_postgres` | 共用 across 的 PostgreSQL，Dify 用獨立 database `dify` | 需驗證 Dify 的 DB migration 能在外部 postgres 正常執行 |
| `redis`       | 共用 across 的 Redis，Dify 用 db=2                   | 需驗證 Celery broker 在非預設 db number 能正常運作     |
| `nginx` | 不需要。across 後端直接走 Docker 內部網路打 `dify-api:5001`，不經過 Caddy | — |


**預計省略的 container（待驗證）：**

以下 container 在完整 Dify 部署中存在，但預期知識庫功能不依賴它們。**尚未實際測試**，需要在開發階段逐一停用並驗證知識庫上傳 + Retrieve API 是否仍正常運作：


| Container       | 預期可省略原因                | 風險                                |
| --------------- | ---------------------- | --------------------------------- |
| `sandbox`       | 程式碼沙箱執行，知識庫功能預期不使用     | 不確定 Dify 啟動時是否強制依賴                |
| `plugin_daemon` | 外掛管理，地端不裝外掛            | 不確定 Dify 1.12.1 啟動是否強制依賴          |
| `ssrf_proxy`    | SSRF 防護（Squid），地端斷網不需要 | 不確定 dify-worker 處理文件時是否走此 proxy   |
| `worker_beat`   | Celery 排程器             | 不確定知識庫的定期任務（如 embedding 狀態更新）是否依賴 |


**可選的 container：**


| Container  | 說明                                                    |
| ---------- | ----------------------------------------------------- |
| `dify-web` | 管理者用的知識庫管理 UI（across 使用者不會接觸，但管理者需要用來設定模型供應商、查看知識庫狀態） |


> **⚠️ 待驗證**：最小部署的實際 container 數量需要在開發階段實測確認。最樂觀是 3 個（dify-api + dify-worker + weaviate，共用 postgres/redis），最保守是 7 個（加回 sandbox、plugin_daemon、ssrf_proxy、worker_beat）。

### 4.4 Dify 與 vLLM 串接

所有模型都跑在地端 vLLM 上，全部透過 LiteLLM 統一路由（across 現有的模型管理層）。目前**全部尚未測試**，模型選型和串接驗證見第十二章統整清單。

```
across / Dify → LiteLLM（統一路由）→ vLLM（地端 GPU）
├── LLM:       LiteLLM → vLLM instance 1  (模型待定)
├── Embedding: LiteLLM → vLLM instance 2  (模型待定，候選 bge-m3)
└── Reranker:  LiteLLM → vLLM instance 3  (模型待定，候選 bge-reranker-v2-m3)
```

Dify 端設定模型供應商指向 LiteLLM（不直接指向 vLLM），across agent-server 也透過 LiteLLM 呼叫 rerank 和 Worker LLM。

### 4.5 共用 PostgreSQL / Redis 注意事項

**PostgreSQL**：Dify 用獨立 database name `dify`，不混在 across 的 `across` database

```yaml
dify-api:
  environment:
    DB_HOST: postgres
    DB_DATABASE: dify        # 獨立 database
    DB_USERNAME: across
```

**Redis**：Dify 用不同的 db number，避免 key 衝突

```yaml
dify-api:
  environment:
    REDIS_HOST: redis
    REDIS_DB: 2              # across 用 db=0
```

---

## 五、資料流

### 5.1 文件上傳

```
User → AIW Copilot 上傳
  → across agent-server
  → Dify API: POST /v1/datasets/{dataset_id}/document/create-by-file
  → Dify Worker: 解析 → 切片 → embedding (via vLLM bge-m3) → 寫入 Weaviate
  → 回傳文件狀態
```

### 5.2 問答檢索（Commander + Worker + Tool）

```
User 在 Commander 對話視窗提問
  → Commander 判斷需要查知識庫
  → 透過 Redis bus 派任務給 kb-rag Worker

      Worker Step 1: Worker LLM 呼叫 kb-rag-retrieve tool
        tool execute 函式裡用程式碼依序執行：
        0. 用 sessionId 從 DB 撈 user 原始 prompt
           （不用 Commander 改寫的 task，確保檢索和 rerank 精準）
        1. 查 RBAC: 這個角色可讀 [KB-A, KB-B]
        2. 用原始 prompt 並行呼叫 Dify Retrieve API:
             POST /v1/datasets/KB-A/retrieve { query: 原始prompt, top_k: 20 } → chunks A
             POST /v1/datasets/KB-B/retrieve { query: 原始prompt, top_k: 20 } → chunks B
        3. 合併 chunks A + B
        4. 用原始 prompt + chunks 呼叫 LiteLLM /rerank → vLLM bge-reranker → 取 top N
        → 回傳 top-K chunks（含來源標註）給 Worker LLM

      Worker Step 2: Worker LLM 看到 top-K chunks → 生成回答
        ⚠️ 回答品質取決於地端 LLM 模型能力：
        - 中文 chunks → 中文回答：需模型支援中文生成
        - 英文 chunks → 中文回答：需模型支援跨語言理解
        - 多個 chunks 的長 context：需模型 context window 夠大
        這些都是模型選型階段要測試的項目（見第八章）

  → Worker 透過 Redis bus 回傳答案給 Commander
  → Commander 轉發給 AIW Copilot
```

> 使用者在現有的 Commander 對話視窗提問即可，不需要獨立的 RAG 頁面或 endpoint。Commander 自動判斷何時派給 kb-rag Worker。Worker 的 LLM model 可獨立設定（`agent.model` 欄位），不影響 Commander 和其他 Worker。

### 5.3 多 KB 檢索與 Rerank 策略

#### 為什麼要合併後才 rerank

across Worker 的做法是**全部 KB 的 chunks 合併後一次 rerank**，不是每個 KB 個別 rerank 再合併：

```
✅ 正確做法（across Worker）：
KB-A 回來 chunks ─┐
KB-B 回來 chunks ─┼→ 合併成一個陣列 → 一次 rerank（原始 prompt vs 全部 chunks）→ top N
KB-C 回來 chunks ─┘

❌ 不能這樣做：
KB-A 回來 chunks → 個別 rerank → top 5
KB-B 回來 chunks → 個別 rerank → top 5  → 合併 15 個
KB-C 回來 chunks → 個別 rerank → top 5
```

個別 rerank 的問題：KB-A 的第 6 名可能比 KB-B 的第 1 名更相關，但在個別 rerank 時已被砍掉。合併後一次 rerank 才能跨 KB 公平比較所有 chunks 的相關性。

**這與 Dify Chatflow 內部邏輯一致。** 查 Dify source code（`knowledge_retrieval_node.py`），Chatflow 的知識檢索節點在 MULTIPLE 模式下也是所有 KB 並行檢索 → 合併全部結果 → 一次 rerank。Dify 只是沒有把這個能力開放在公開的 Retrieve API 上（`POST /v1/datasets/{dataset_id}/retrieve` 一次只能查一個 dataset），所以 across 需要在 Worker 端自己實作這段。

#### 併發呼叫與 top_k 控制

**併發**：across Worker 用 Promise.all 同時呼叫所有 KB 的 Dify Retrieve API，延遲等於最慢的那個 KB，不是加總。Dify 自建版沒有 API rate limit（rate limit 只在 `BILLING_ENABLED=True` 的雲端版才啟用），併發數量不受限制。

**top_k 控制**：如果角色可讀的 KB 數量多，每個 KB 都回 top_k=20 會導致送進 rerank 的 chunks 總量很大（5 個 KB = 100 chunks），增加 reranker 延遲。建議採用動態 top_k：

```
總 chunk 預算 = 40（可設定）
角色可讀 KB 數 = N
每個 KB 的 top_k = Math.ceil(40 / N)

例：2 個 KB → 每個 top_k=20，合計 40
例：5 個 KB → 每個 top_k=8，合計 40
```

這樣不管幾個 KB，送進 reranker 的總量固定在預算內，避免延遲隨 KB 數線性增長。

> **⚠️ 動態 top_k 未必會做。** 這只是一種策略，實際要看測試結果。也可能每個 KB 就是固定 top_k（例如固定 12），看哪種效果好再決定。最終的 top_k 策略（動態 or 固定）需要在測試階段根據 reranker 延遲和回答品質來定。

**已確認 Dify Retrieve API 支援 per-request top_k 覆蓋。** 查 Dify source code（`hit_testing_service.py` line 43-73），知識庫建立時設的 top_k 只是預設值，每次 API 呼叫時在 `retrieval_model` 參數裡帶 `top_k` 就能覆蓋：

```json
POST /v1/datasets/{dataset_id}/retrieve
{
  "query": "原始 prompt",
  "retrieval_model": {
    "search_method": "hybrid_search",
    "top_k": 8,
    "score_threshold_enabled": true,
    "score_threshold": 0.5,
    "reranking_enable": false
  }
}
```

`reranking_enable` 設為 `false`，因為 rerank 由 across Worker 合併所有 KB chunks 後統一做，不在 Dify 端個別做。

---

## 六、across 端現有能力 vs 需要新開發的功能

### 6.1 現有能力盤點

以下是 across agent-server **目前已有**、可部分復用的程式碼：


| 現有元件           | 檔案                                     | 能力                                                       | 能否直接用於 RAG                                                                                                                            |
| -------------- | -------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Chat handler | `chat/handler.ts` | Vercel AI SDK `streamText()` streaming 回應 | **不需要改**。Commander 透過 `assign-worker` tool 派任務給 Worker，Worker 透過 tool calling 呼叫 `kb-rag-retrieve` tool 取得 chunks 後生成回答。現有的 Commander + Worker + tool 架構可直接使用 |
| kb-search tool | `tools/kb-rag.ts`                      | 呼叫 `hybridSearch()` 回傳 chunks 給 LLM agent                | **不能直接用**。是 tool calling 機制（LLM 隱式決定是否呼叫），不是主動的 RAG pipeline。且全英文、綁死 pgvector                                                         |
| rerank 函式      | `services/rag-service.ts:166-210`      | 呼叫 LiteLLM `/rerank` endpoint，回傳重排後的結果                   | **邏輯可復用但要改**。目前寫死 `model: 'rerank-v3.5'`（Cohere 雲端 API），地端斷網無法使用。需要改成指向 vLLM 的本地 bge-reranker model endpoint，model name 和 API 格式可能需調整 |
| LLM 呼叫         | `lib/llm.ts`                           | 透過 Vercel AI SDK `createOpenAI()` 連 LiteLLM，支援 streaming | **可復用**。呼叫 LLM 生成回答可用此 provider                                                                                                       |
| RBAC 系統        | `agents/prompt-builder.ts` + DB schema | 依 user 角色決定可用的 agent 和 tool                              | **不能直接用**。現有 RBAC 管的是 agent/tool 權限，沒有「角色 → 知識庫」的 mapping                                                                             |
| Embedding 呼叫   | `memory/embedding.ts`                  | 呼叫 LiteLLM embedding endpoint                            | **不需要 across 自己算**。embedding 由 Dify Worker 處理。但 across 需要控制上傳量（單次上傳檔案數、單檔大小上限），避免大量檔案同時進入 Dify Worker 的 embedding queue 壓垮地端 vLLM     |


### 6.2 across 完全沒有以及需要從零開發的功能

#### A. 設定層（不需寫程式碼，但需要設定和驗證）


| #   | 功能                | 位置                       | 性質     | 說明                                                                                                          |
| --- | ----------------- | ------------------------ | ------ | ----------------------------------------------------------------------------------------------------------- |
| A1  | Docker Compose 設定 | `docker-compose.yml`     | 設定     | 新增 dify-api、dify-worker、weaviate service 定義，設定 depends_on、healthcheck、環境變數                                  |
| ~~A2~~ | ~~Caddy 反向代理路由~~ | — | — | 短期不需要。across 後端直接走 Docker 內部網路打 Dify API，不經 Caddy。Dify Web UI 的 Caddy 路由留到開發/測試階段需要時再設定 |
| A3  | 環境變數              | `apps/agent-server/.env` | 設定+程式碼 | 新增 `DIFY_SERVICE_URL`、`DIFY_KB_API_KEY` 等，加入 `env.ts` Zod schema                                            |
| A4  | LiteLLM 設定        | `litellm_config.yaml`    | 設定     | 新增 vLLM 本地模型 endpoint（LLM + Embedding + Reranker）                                                           |
| A5  | Dify 模型供應商設定      | Dify Web UI 操作           | 設定     | 在 Dify 管理介面設定模型供應商指向 LiteLLM。Dify 上傳文件時需要呼叫 Embedding model 算向量，這個設定告訴 Dify 去哪裡呼叫 Embedding（LiteLLM → vLLM） |


#### B. 資料庫層（需要寫 schema + migration）


| #   | 功能                     | 位置                        | 性質   | 說明                                              |
| --- | ---------------------- | ------------------------- | ---- | ----------------------------------------------- |
| B1  | 知識庫 metadata table     | `packages/db/src/schema/` | 從零開發 | 記錄 across 管理的知識庫（對應 Dify dataset_id、名稱、描述、建立者等） |
| B2  | 角色 ↔ 知識庫 mapping table | `packages/db/src/schema/` | 從零開發 | 記錄哪些角色可以存取哪些知識庫（多對多關聯）                          |
| B3  | Drizzle migration      | `packages/db/drizzle/`    | 從零開發 | 產生並執行 B1、B2 的 DB migration                      |


#### C. API 層 — api-admin（知識庫管理，從零開發）


| #   | 功能          | 位置                                                          | 性質                            | 說明                                                                                         |
| --- | ----------- | ----------------------------------------------------------- | ----------------------------- | ------------------------------------------------------------------------------------------ |
| C1  | 建立知識庫       | `POST /api/internal/knowledge-bases`                        | 串接 Dify API + across metadata | across 建立知識庫 metadata → 呼叫 Dify API（`POST /v1/datasets`）建立 dataset → 儲存 dataset_id mapping |
| C2  | 列出知識庫       | `GET /api/internal/knowledge-bases`                         | 串接 Dify API + across metadata | 從 across DB 列出知識庫（含角色權限資訊），可搭配 Dify API 取得文件數等狀態                                           |
| C3  | 刪除知識庫       | `DELETE /api/internal/knowledge-bases/:id`                  | 串接 Dify API + across metadata | 刪除 across metadata + 呼叫 Dify API（`DELETE /v1/datasets/{id}`）刪除 dataset                     |
| C4  | 上傳文件到知識庫    | `POST /api/internal/knowledge-bases/:id/documents`          | 串接 Dify API                   | 接收 multipart/form-data → 轉發到 Dify API 上傳文件（切片 + embedding 由 Dify 處理）                       |
| C5  | 列出知識庫文件     | `GET /api/internal/knowledge-bases/:id/documents`           | 串接 Dify API                   | 呼叫 Dify API 列出 dataset 下的文件                                                                |
| C6  | 刪除文件        | `DELETE /api/internal/knowledge-bases/:id/documents/:docId` | 串接 Dify API                   | 呼叫 Dify API 刪除文件                                                                           |
| C7  | 綁定角色到知識庫    | `POST /api/internal/knowledge-bases/:id/roles`              | 從零開發                          | 更新角色 ↔ KB mapping table（Dify 沒有這個功能）                                                       |
| C8  | 查詢角色可存取的知識庫 | `GET /api/internal/knowledge-bases/by-role/:roleId`         | 從零開發                          | 查詢特定角色可讀的知識庫列表（Dify 沒有這個功能）                                                                |


#### D. agent-server — 新增 kb-rag Worker agent

新增一個 `kb-rag` Worker agent，融入現有的 Commander + Worker 架構。Commander 判斷使用者問題需要查知識庫時，透過 Redis bus 派任務給 kb-rag Worker。**Worker 用自己的 LLM model 生成回答**，model 可獨立設定，不影響 Commander 和其他 Worker。

**與現有 kb-search tool 的差別：**


|           | 現有 kb-search tool                    | 新的 kb-rag Worker                                        |
| --------- | ------------------------------------ | ------------------------------------------------------- |
| 類型        | Commander 的 tool（Commander 自己回答）     | 獨立的 Worker agent（Worker 自己回答）                           |
| LLM model | 用 Commander 的 model                  | **用 Worker 自己的 model**（可獨立設定）                           |
| 資料來源      | pgvector（across 本地）                  | Dify Retrieve API（可查多個 KB）                              |
| RBAC      | 無（所有人查同一個 KB）                        | 有（依角色決定查哪些 KB）                                          |
| 關鍵字搜尋     | `to_tsvector('english', ...)`，中文不能用  | Dify 內部處理（jieba 中文分詞）                                   |
| Rerank    | 無（RRF 排序後直接回傳）                       | 有（呼叫 vLLM bge-reranker）                                 |
| 回答方式 | 回傳 chunks 給 Commander，Commander 生成回答 | **Worker LLM 呼叫 tool 取得 chunks → Worker LLM 生成回答 → 回傳給 Commander LLM → Commander LLM 回覆 User**（共 4 次 LLM：Commander 2 次 + Worker 2 次） |


**Worker 需要開發的功能：**


| #   | 功能                       | 位置                              | 說明                                                                                                                                                                                                                               |
| --- | ------------------------ | ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | kb-rag-retrieve tool 定義  | `tools/kb-rag-retrieve.ts`（新檔）  | 定義 Vercel AI SDK tool（description、inputSchema、execute），Worker LLM 呼叫此 tool 觸發整個 RAG 檢索流程。tool 的 execute 函式內部依序呼叫 D2-D5                                                                                                           |
| D2  | 取得 user 原始 prompt        | tool execute 內呼叫                | Commander 派任務時帶的 `task` 是 Commander LLM 改寫過的描述，不一定是 user 原始問題。但 rerank 和 Dify Retrieve 都需要 user 原始 prompt 才能精準檢索和排序。**解法**：`assign-worker` 已帶 `sessionId`，Worker 用 sessionId 從 DB 撈最後一條 user message，取得原始 prompt（程式碼保證，不靠 LLM）   |
| D3  | Dify Retrieve API client | `services/dify-retrieve.ts`（新檔） | 封裝 Dify `POST /v1/datasets/{id}/retrieve` 呼叫，處理認證、逾時、錯誤、回傳型別定義                                                                                                                                                                   |
| D4  | RBAC 知識庫權限查詢             | `services/kb-rbac.ts`（新檔）       | 查 DB：這個 user 的角色可以存取哪些知識庫 → 回傳 dataset_id 列表                                                                                                                                                                                     |
| D5  | 多 KB 並行檢索 + 合併           | `services/kb-retrieval.ts`（新檔）  | 用 D2 取得的原始 prompt，對每個允許的 KB 並行呼叫 D3（Promise.all，每個 KB 取 top_k 個 chunks）→ 合併所有 chunks 到一個陣列                                                                                                                                       |
| D6  | Rerank                   | `services/kb-retrieval.ts`      | 用**原始 prompt** + 合併後的 chunks，呼叫 LiteLLM `/rerank` endpoint → 取 top N。復用現有 `rag-service.ts` 的 rerank 邏輯，但需改 model name 指向 vLLM bge-reranker（現有寫死 `rerank-v3.5` 是 Cohere 雲端，地端不能用）。tool execute 回傳 top-K chunks（含來源標註）給 Worker LLM |
| D7  | Worker agent 定義          | `agents/kb-rag/agent.md`（新檔）    | 定義 agent YAML frontmatter：id、name、role: worker、model（地端 LLM）、requiredFeature 等                                                                                                                                                   |
| D8  | RAG prompt template      | Worker 的 `agent.md` prompt body | Worker 的 system prompt，指示 LLM 看到 tool 回傳的 chunks 後根據內容回答、標註來源、處理無相關資料的情況，支援中英文                                                                                                                                                   |


**執行流程（共 4 次 LLM）：**

```
Commander LLM Step 1: 判斷需要查知識庫 → 呼叫 assign-worker 派任務給 kb-rag Worker
  Worker LLM Step 1: 呼叫 kb-rag-retrieve tool
    tool execute 內部跑 D2→D3→D4→D5→D6（全部程式碼，不經 LLM）
    → 回傳 top-K chunks
  Worker LLM Step 2: 看到 chunks → 生成回答 → 回傳給 Commander
Commander LLM Step 2: 收到 Worker 回答 → 回覆 User
```

不需要修改 `execute.ts`，用現有的 Worker + tool 架構。但一次 RAG 問答共 4 次 LLM 呼叫（Commander 2 次 + Worker 2 次），相關影響見第九章雙 LLM 問題。

**Worker 執行限制（現有設定）：**


| 限制                  | 現有值                                            | 對 kb-rag Worker 的影響                                                                                                                                                               |
| ------------------- | ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Worker timeout      | 15 分鐘（`WORKER_TIMEOUT_MS = 900_000`）           | 從 Commander 派任務到 Worker 回傳答案的上限。KB 數量多、文件量大時，並行 Retrieve + 合併大量 chunks + rerank + LLM 生成回答的總延遲可能接近甚至超過此值                                                                          |
| Commander max steps | 10 步（`DEFAULT_MAX_STEPS = 10`，chat/handler.ts） | Commander 自己的步數上限。由 Vercel AI SDK 的 `stepCountIs` 計算，每次 LLM 呼叫算一步。Commander 要先花步數判斷使用者意圖、決定派給哪個 Worker（呼叫 `assign-worker`）、收到結果後處理回覆。如果一個 prompt 涉及多個 Worker（例如先查知識庫再查 ELK），步數會更多 |
| Worker max steps | 30 步（`DEFAULT_MAX_STEPS = 30`，execute.ts，獨立計算） | Worker 自己的步數上限。kb-rag Worker 使用 tool calling 方式：Step 1 呼叫 tool + Step 2 生成回答 = Worker 2 步。加上 Commander 的 2 步，一次 RAG 問答共 4 次 LLM 呼叫（Commander 2 + Worker 2），相關影響見第九章雙 LLM 問題 |


> **⚠️ rerank 是否應計入步數 — 待討論。** 目前規劃 rerank 用程式碼直接 `fetch()` 打 LiteLLM `/rerank` endpoint，不經過 Vercel AI SDK 的 `streamText()`，所以 SDK 不會將 rerank 計入步數。這代表 rerank 不受步數限制保護 — 如果 rerank 卡住或很慢，步數機制不會喊停，只有 timeout 會。如果要讓 rerank 被步數管控，需要改成 Vercel AI SDK tool calling 的方式實作，但步數就會多算。此議題需在實作階段討論決定。
>
> **⚠️ timeout 是否足夠需要在測試階段驗證。** 影響 timeout 的主要因素：KB 數量、每個 KB 的文件量和 chunks 數量、向量庫的查詢效能、reranker 處理大量 chunks 的延遲、地端 LLM 的推論速度。這些因素與**中期的向量庫選型**直接相關 — 向量庫的能力決定了能承載多少文件/chunks、每次 Retrieve 的回應時間，進而影響整條 pipeline 是否會超過 timeout。短期先用現有的 15 分鐘設定，測試階段觀察實際延遲再決定是否需要調整。

#### E. 前端 — web（AIW Copilot 頁面）


| #   | 功能      | 位置              | 說明                                                    |
| --- | ------- | --------------- | ----------------------------------------------------- |
| E1  | 知識庫管理頁面 | `apps/web/app/` | MIS 專用頁面：列出知識庫、建立/刪除知識庫、上傳文件、查看文件清單、綁定角色              |
| E2  | 文件上傳元件  | `apps/web/app/` | drag & drop 上傳、進度顯示、支援格式提示（.md / .docx / .pdf / .txt） |
| E3  | 來源引用顯示  | `apps/web/app/` | Commander 回答中引用知識庫內容時，顯示來源（KB 名稱、文件名、相關段落預覽）          |


> **不需要獨立的 RAG 問答介面**，使用者直接在現有的 Commander 對話視窗提問，Commander 自動判斷是否需要查知識庫。

### 6.3 改動總覽


| 類別               | 數量       | 從零開發                  | 設定/改寫                      |
| ---------------- | -------- | --------------------- | -------------------------- |
| A. 設定層           | 5 項      | 0                     | 5（設定檔修改 + Dify UI 操作）      |
| B. 資料庫           | 3 項      | 3                     | 0                          |
| C. 知識庫管理 API     | 8 項      | 2（C7、C8 角色綁定，Dify 沒有） | 6（C1-C6 串接 Dify 現有 API）    |
| D. kb-rag Worker | 8 項      | 全部需要開發                | 新增 3 個 ts 檔 + 1 個 agent.md |
| E. 前端            | 3 項      | 3                     | 0                          |
| **合計**           | **27 項** | **13 項從零開發**          | **14 項設定/改寫/串接/邏輯補充**      |


---

## 七、向量庫

### 短期：沿用 Weaviate

Dify 本地環境已在 Weaviate 1.27.0 上運行，短期不更換，避免增加變數。

### 中期：向量庫選型評估

Dify 支援 30+ 種向量庫，選擇不被鎖死。但切換向量庫**不等於自由切換**，需要評估：


| 評估項目           | 說明                                 |
| -------------- | ---------------------------------- |
| embedding 維度相容 | 新舊向量庫是否支援相同維度，不相容需 re-embed 全部文件   |
| 索引演算法差異        | HNSW vs IVF vs DiskANN，檢索品質和效能可能不同 |
| 中文全文搜尋支援       | 不是每個向量庫都有 BM25 / 全文搜尋              |
| 資源佔用           | 各向量庫的 RAM / CPU / 磁碟需求差異大          |
| 資料遷移           | 可能需要匯出 → re-embed → 匯入             |


中期候選（全部需要測試和評估）：


| 候選       | 適合場景                  | 需要測試和評估的項目                               |
| -------- | --------------------- | ---------------------------------------- |
| Qdrant   | 輕量、百萬級、Rust、mmap 省記憶體 | 中文檢索品質、與 Dify 的相容性、遷移成本、地端 VM 資源佔用       |
| Milvus   | 千萬級、分散式               | 資源佔用、部署複雜度（需 etcd + minio）、地端是否 overkill |
| Weaviate | 目前使用中                 | 大量文件下的效能、RAM 佔用增長、長期是否滿足需求               |
| pgvector | 共用現有 postgres         | 與 across 應用資料搶資源的影響、大量向量搜尋對 DB 效能的衝擊     |


---

## 八、模型選型（短期評估項目）

整個 RAG 流程涉及三種模型，各自在不同階段使用，全部需要在地端用 vLLM 跑，都需要評估測試。

### 8.1 RAG 流程中每個模型的角色

```
文件上傳階段（Dify Worker 處理，一次性）：
  文件 → 切片 → [Embedding Model] 把每個 chunk 轉成向量 → 存入 Weaviate
                 ↑ 模型 1（上傳時呼叫，一次性）

使用者問答階段（每次提問都會觸發，三個模型都會被呼叫）：
  使用者問題 → [Embedding Model] 把問題轉成向量 → Weaviate 向量搜尋 → chunks
               ↑ 模型 1（每次問答都會呼叫，由 Dify Retrieve API 內部處理，across 看不到）

  chunks → [Reranker Model] 重新排序 → top N chunks
            ↑ 模型 2（每次問答都會呼叫，由 across Worker 呼叫 LiteLLM）

  top N chunks + 問題 → [LLM] 生成回答
                         ↑ 模型 3（每次問答都會呼叫，由 across Worker 自己的 LLM）
```

### 8.2 三種模型的用途和評估重點


| 模型類型          | 什麼時候用                                       | 誰呼叫                                         | 頻率           | 評估重點                                                                                                                                                               |
| ------------- | ------------------------------------------- | ------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Embedding** | 上傳文件時算向量 + 每次問答時算查詢向量 | Dify（上傳時）+ Dify Retrieve API（查詢時） | 上傳：一次性；查詢：每次 | 中英文語義匹配品質、維度大小、embedding 速度、VRAM 佔用、DRAM 佔用 |
| **Reranker**  | 每次問答，對合併後的 chunks 重新排序                      | across agent-server → LiteLLM → vLLM        | 每次問答         | 中英文重排品質、延遲（直接加到回答時間）、VRAM 佔用                                                                                                                                       |
| **LLM**       | 每次問答，Commander LLM 看到 tool 回傳的 chunks 後生成回答 | Commander（現有 streamText 機制）→ LiteLLM → vLLM | 每次問答         | **關鍵驗證項目**：(1) 中文 chunks → 中文回答 (2) 英文 chunks → 中文回答 (3) 多個 chunks 的長 context 理解能力 (4) 能否正確引用來源 (5) VRAM 佔用、推論速度。現有 Commander 用雲端 Claude 可以做到，但地端 LLM 能否達到相同品質需要測試 |


### 8.3 across 現有能力 vs 需要新增


| 模型類型          | across 現在有沒有能力呼叫                                                                                                                                                                         | 需要改什麼                                                                                                |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| **Embedding** | **不需要 across 呼叫** — Dify 內部處理（上傳時 Worker 呼叫、查詢時 Retrieve API 內部呼叫），across 不介入                                                                                                            | 只需在 Dify Web UI 設定 Embedding model 指向 vLLM endpoint                                                  |
| **Reranker**  | **有程式碼但不能直接用** — `rag-service.ts:166-210` 有 rerank 函式，但寫死 `model: 'rerank-v3.5'`（Cohere 雲端 API），地端斷網無法使用                                                                                 | 需修改 model name 指向 vLLM 的本地 bge-reranker，確認 vLLM 的 rerank API 格式是否與現有程式碼相容（可能需調整 request/response 格式） |
| **LLM** | `lib/llm.ts` 有 Vercel AI SDK provider 可呼叫 LiteLLM → LLM，`chat/handler.ts` 有 streaming 回應 | **不需要額外寫 LLM 呼叫邏輯**。kb-rag Worker LLM 呼叫 `kb-rag-retrieve` tool 取得 chunks 後，Worker LLM 自己生成回答（Step 2），走現有的 `execute.ts` → `streamText()` 流程。只需在 Worker 的 `agent.md` 設定 model 指向地端 vLLM |


### 8.4 候選模型（全部待測試，尚未決定）

> **⚠️ 以下所有模型都是候選清單，尚未確定任何一個。** 每個模型都需要驗證：(1) 能否在地端 vLLM/TEI 上正常運行 (2) 授權條款是否允許商用 (3) 中英文實際品質。本文件其他章節提到的具體模型名稱（如 bge-m3、bge-reranker-v2-m3 等）也都是候選，不是最終決定。

> **優先測試非中資模型。**

**Embedding（模型 1）：**

決定文件向量化和語義搜尋的品質。選錯的話相關文件搜不到，後面 rerank 和 LLM 都救不回來。


| 模型                    | 中資    | 來源                                | 維度   | 大小                           | 中英文能力      | 備註              |
| --------------------- | ----- | --------------------------------- | ---- | ---------------------------- | ---------- | --------------- |
| Jina Embeddings v5    | 否     | Jina AI（德國，已被 Elastic 收購）         | 1024 | ~~0.7GB（small）/~~0.2GB（nano） | 好（119+ 語言） | 優先測試，2026-02 發布 |
| multilingual-e5-large | 否     | Microsoft Research（美國，研發在北京 MSRA） | 1024 | ~2.2GB                       | 好          | 優先測試            |
| Nomic Embed Text      | 否     | Nomic AI（美國紐約）                    | 768  | ~0.5GB                       | 中等         | 輕量備選            |
| BGE-M3                | **是** | BAAI 北京智源（中國北京，政府資助）              | 1024 | ~2.2GB                       | 極好（中英日韓）   | 中文最強但中資         |
| bge-large-zh-v1.5     | **是** | BAAI 北京智源                         | 1024 | ~1.3GB                       | 中文極好，英文中等  | 中資              |


> **注意**：Embedding model 一旦選定，已上傳的文件都是用這個 model 算的向量。換 model = 全部文件要重新 embedding。

**Reranker（模型 2）：**

決定 chunks 排序品質，影響 LLM 看到的 context 是否精準。每次問答都會呼叫，延遲直接加到回答時間上。

> **優先測試非中資模型。**


| 模型                 | 中資    | 來源                        | 大小     | 中英文能力           | 備註                            |
| ------------------ | ----- | ------------------------- | ------ | --------------- | ----------------------------- |
| Jina Reranker v3   | 否     | Jina AI（德國，已被 Elastic 收購） | ~0.6GB | 好（131K context） | 優先測試，2025-10 發布，基於 Qwen3-0.6B |
| bge-reranker-v2-m3 | **是** | BAAI 北京智源                 | ~2.2GB | 極好（多語系）         | 中文最強但中資                       |
| bge-reranker-large | **是** | BAAI 北京智源                 | ~1.3GB | 好               | 中資                            |


**LLM（模型 3）：**

決定最終回答品質。需要能理解中英文 context、正確引用來源、以使用者語言回答。

> **優先測試非中資模型。**


| 模型                | 中資    | 來源                     | 參數量                      | VRAM（Q4）     | 中英文能力    | 備註                             |
| ----------------- | ----- | ---------------------- | ------------------------ | ------------ | -------- | ------------------------------ |
| Llama 4 Scout     | 否     | Meta（美國）               | 109B total / 17B active  | ~12GB        | 好（多模態）   | 優先測試，2025-04 發布，MoE 架構         |
| Mistral Small 4   | 否     | Mistral AI（法國）         | 119B total / 6-8B active | ~6GB         | 英文好，中文中等 | 優先測試，2026-03 發布，MoE，Apache 2.0 |
| Gemma 4           | 否     | Google DeepMind（美國）    | 31B Dense / 26B MoE      | ~20GB / ~5GB | 好        | 優先測試，2026-04 發布，Apache 2.0     |
| Phi-4             | 否     | Microsoft（美國）          | 14B                      | ~10GB        | 英文好，中文中等 | 優先測試，2025-01 發布                |
| Qwen 3.6          | **是** | 阿里雲（中國杭州）              | 27B Dense / 35B MoE      | ~18GB / ~5GB | 極好       | 中文最強但中資，2026-04 發布             |
| DeepSeek V4 Flash | **是** | DeepSeek 深度求索（中國杭州）    | 284B total / 13B active  | ~10GB        | 好        | 中資，2026-04 發布，MIT              |
| Yi                | **是** | 01.AI 零一萬物（中國北京，李開復創辦） | 6B/34B                   | 5/22GB       | 好        | 中資                             |


### 8.5 資源佔用分析（全部跑在同一台 VM）

across、Dify、vLLM、Weaviate 全部跑在同一台客戶 VM 裡，搶的是同一台機器的 VRAM / DRAM / CPU / 磁碟。

#### 模型常駐 VRAM（模型載入後不釋放）


| 模型               | VRAM 預估   | 備註              |
| ---------------- | --------- | --------------- |
| Embedding（候選，待定） | ~0.5-2GB  | 視模型大小           |
| Reranker（候選，待定）  | ~0.5-2GB  | 視模型大小           |
| LLM（候選，待定）       | ~6-48GB   | 視模型參數量和量化方式     |
| **合計**           | **待測試確認** | 需在測試階段用實際候選模型量測 |


> 三個模型可能需要分別跑不同的 vLLM instance（不同 port），因為 vLLM 一個 instance 只能載一個模型。Embedding 也可以考慮用 TEI（Text Embeddings Inference）替代 vLLM，效能更好。這些都是評估階段要確認的。

#### 文件上傳索引時的資源消耗（Embedding + 向量資料庫同時吃資源）

上傳文件做索引時，Embedding 模型和 Weaviate **同時在消耗資源**，不是做完一個才做下一個：

```
Dify Worker 把 chunks 分批送給 vLLM embedding
  ↓
vLLM embedding 算出向量（持續佔用 GPU）
  ↓                    ← 這兩個同時在跑
向量送到 Weaviate 寫入 + 建 HNSW 索引（持續佔用 DRAM + CPU）
```


| 資源     | Embedding 模型                    | 向量資料庫（Weaviate）                                | 同時跑的影響       |
| ------ | ------------------------------- | ---------------------------------------------- | ------------ |
| VRAM   | 模型常駐 + 每批 chunks 的中間計算          | 不佔 VRAM                                        | Embedding 獨佔 |
| DRAM   | 批次 chunks 的 input/output buffer | **HNSW 索引建構吃 DRAM**，向量越多索引越大，索引期間 DRAM 使用量持續增長 | 互搶 DRAM      |
| CPU    | tokenization（前處理）               | **HNSW 索引建構吃 CPU**，每插入一個新向量要計算跟現有向量的距離         | 互搶 CPU       |
| 磁碟 I/O | 不大                              | 索引寫入磁碟                                         | 互搶 I/O       |


Weaviate 的 HNSW 索引建構不是單純「收到向量存起來」，每插入一個新向量都要計算距離 + 更新索引圖結構，向量越多每次插入的計算量越大。大量文件上傳時 DRAM 壓力來自兩邊同時增長。這也是為什麼上傳量需要控制（第十二章 E1-E4）。

> **⚠️ 以上所有資源數據都是預估，需要在測試階段用實際候選模型和實際文件量在目標硬體上量測。** 模型的實際 VRAM 佔用、Weaviate 的 DRAM 增長曲線、reranker 延遲、LLM 推論速度都需要實測確認（見第十二章 A、E 類測試項目）。

---

## 九、優缺點

### 優點

1. **向量庫彈性** — Dify 支援 30+ 種，中期選型評估不被鎖死
2. **中文開箱即用** — Dify 內建 jieba 分詞
3. **沿用 Weaviate** — Dify 現有環境直接用，短期不多加 container
4. **RBAC 完全自主** — across 控制，不受 Dify 限制

### 缺點

1. **Retrieve API 一次只能查一個 KB** — across 要並行打 N 次 + 自己合併
2. **Rerank 要 across 自己寫** — Dify Retrieve API 不含 rerank
3. **組 context + LLM 呼叫要新寫** — 現有 agent-server 沒有這段邏輯
4. **多一套 Dify 服務** — 版本升級、設定同步需維護
5. **多次 LLM 問題** — 一次問答會過 4 次 LLM（見下方說明），拖慢速度且可能掉資訊

#### 雙 LLM 問題說明

目前的 Commander + Worker + Tool 架構下，一次 RAG 問答會經過 **4 次 LLM**：

```
User 提問
  → Commander LLM（第 1 次）：判斷要派給哪個 Worker → 呼叫 assign-worker
  → kb-rag Worker LLM（第 2 次）：呼叫 kb-rag-retrieve tool
      tool 內部跑程式碼（不經 LLM）
      → 回傳 top-K chunks
  → kb-rag Worker LLM（第 3 次）：看到 chunks → 生成回答 → 回傳給 Commander
  → Commander LLM（第 4 次）：收到 Worker 回答 → 回覆 User（可能會再加工）
```

**影響：**


| 問題       | 說明                                                                                                               |
| -------- | ---------------------------------------------------------------------------------------------------------------- |
| **速度**   | 至少 2 次 LLM 呼叫（Commander 判斷 + Worker 回答），如果 Commander 收到 Worker 答案後又加工則是 3 次。每次 LLM 呼叫在地端 vLLM 可能需要數秒，加總會明顯拖慢回答速度 |
| **資訊遺失** | Worker 生成的回答經過 Commander 轉發時，Commander LLM 可能會摘要、改寫、或遺漏 Worker 回答中的細節（例如來源引用、數據、專業術語）                            |
| **語意曲解** | Commander LLM 對 Worker 回答的理解可能不完全正確，尤其是專業領域的內容（資安術語、法規條文），轉述時可能曲解原意                                              |


**這是 Commander + Worker 架構在 RAG 場景下的固有代價。** 需要在測試階段觀察實際影響程度，如果問題嚴重，可考慮的緩解方案：


| 緩解方案              | 白話說明                                                                         | 代價                                                  |
| ----------------- | ---------------------------------------------------------------------------- | --------------------------------------------------- |
| Commander 透傳      | 在 Commander 的 system prompt 裡寫「收到 kb-rag Worker 的回答，請原封不動回覆給使用者，不要改寫」        | LLM 不一定聽話，可能還是會自己改寫內容                               |
| 獨立 endpoint（做法 B） | 不走 Commander，另外開一個專用的 RAG API。使用者問知識庫的問題直接打這個 endpoint，只過 Worker 一次 LLM 就回答  | 失去 Commander 的多 tool 協作能力（例如不能同時查知識庫 + 查 ELK log）   |
| Worker 回答標記       | Worker 回答時在內容裡加一個標記，across 的程式碼看到標記就直接把 Worker 的回答丟給前端，不再經過 Commander LLM 處理 | 需要修改 bus-runner 程式碼邏輯，改變現有 Commander + Worker 的回傳機制 |


> **短期不解決。** 這個雙 LLM 問題是短期 Commander + Worker 架構的固有代價。長期架構不確定是否會沿用同樣的模式，所以短期先記錄問題，在測試階段觀察實際影響程度（速度慢多少、資訊掉多少），再決定是否需要緩解以及用哪個方案。

---

## 十、文件格式

### 文件格式

**短期支援格式：** `.md`、`.docx`、`.pdf`、`.txt`（Dify 原生支援，across 前端只需設定 accept，零額外開發成本）

**短期不支援：**

- 圖片（文件內的圖片會被跳過，不會被知識庫收錄）
- 掃描件 PDF（沒有文字層的 PDF 無法提取內容，需要 OCR，Dify 基本解析器不做 OCR）
- 表格解析可能會丟失資料（簡單表格勉強能用，合併儲存格、多層表頭、巢狀表格的結構會被打散）

**前端上傳頁面需顯示提示：**

- 文件內的圖片不會被收錄，重要的圖表內容請用文字描述補充在文件中
- 掃描件 PDF 不支援，請上傳電腦產生的 PDF（有文字層）
- 複雜表格可能結構丟失，建議將重要表格內容用文字說明

**中期再評估的格式：** Excel（XLSX/CSV）需要特殊處理：


| 問題       | 說明                                        |
| -------- | ----------------------------------------- |
| 表格結構被破壞  | 切片器把 Excel 當純文字處理，欄位名和欄位值的對應關係在切片後丟失      |
| 分頁資訊消失   | Sheet name 和分頁邊界不一定會保留                    |
| 數字欄位語義差  | embedding 對數字的語義理解很弱，整欄數字的 chunk 向量搜尋幾乎沒用 |
| 合併儲存格、公式 | 轉純文字後完全丟失                                 |


如果未來要支援 Excel，建議先將表格轉成結構化的描述性文字再上傳，而非直接丟原始檔案。

---

## 十一、中長期規劃（待討論）

以下項目不在短期 2 週範圍內，記錄下來供後續討論。

### 中期


| 項目                        | 說明                                                                                                                                     |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| 各部門自管知識庫                  | 開放各部門角色（如部門 Admin）自行新刪修查自己部門的知識庫，不再全部由 MIS 統一管理                                                                                        |
| 向量庫選型評估                   | 評估 Qdrant、Milvus、Weaviate 等向量庫在地端場景的效能、資源佔用、中文檢索品質，決定是否從 Weaviate 切換                                                                   |
| Dify Retrieve API 限制的替代方案 | 評估是否需要繞過 Dify 的單 KB per-request 限制（例如用 Dify 的 Console API 或自建 retrieval service）                                                       |
| Embedding model 換型的遷移策略   | 如果評估後需要換 embedding model，制定全量文件 re-embedding 的流程和時間估算                                                                                  |
| Excel 格式支援                | Excel（XLSX/CSV）需要特殊處理（表格結構轉換），不能直接丟原始檔案。短期已支援 .md / .docx / .pdf / .txt                                                                |
| 向量資料庫轉換                   | 如果向量庫選型評估後決定從 Weaviate 切換到其他向量庫（Qdrant、Milvus 等），需制定資料遷移流程（匯出 → re-embed → 匯入）、停機時間評估、回滾方案                                             |
| 閒置資源釋放 | 知識庫模型（Embedding / Reranker / LLM）和向量資料庫佔用主機資源（VRAM / DRAM / SSD）但沒在使用時，應如何釋放資源空間。例如：模型從 GPU 卸載、向量庫索引從記憶體卸載、vLLM instance 動態啟停等策略的規劃和測試 |
| 多 across 實例資源測試 | 一台物理主機上跑多個 VM、每個 VM 一套 across + 知識庫，測試知識庫的 VRAM / DRAM / SSD / GPU 資源佔用在多租戶場景下的互搶影響和上限。需搭配向量庫選型評估一起做 |


### 長期


| 項目 | 說明 |
|------|------|
| 自建 RAG 替換 Dify | 根據使用回饋決定是否用 LlamaIndex / 自建 rag-ingestion 服務取代 Dify，完全掌控 RAG pipeline。自建時可實作 Parent-Child Chunking（搜 child → 取回 parent 的階層式切片，提升長文件檢索精度） |
| 知識庫備份機制         | 向量資料庫（Weaviate 或其他）的備份策略、備份頻率、還原流程、備份檔大小評估                                                                                               |
| 多主機知識庫擴充        | 客戶購買多台主機時，知識庫應如何跨主機擴充。例如：向量庫是否需要分散式部署、知識庫資料是否需要同步或分區、哪些向量庫支援叢集模式                                                                         |


---

## 十二、待測試 / 待驗證項目統整

以下是散落在文件各章節的所有待測試、待驗證、待討論項目，統整在此方便追蹤。

### A. 模型相關（全部跑地端 vLLM，透過 LiteLLM 路由）


| #   | 項目                    | 測試內容                                                                                                                   | 相關章節       |
| --- | --------------------- | ---------------------------------------------------------------------------------------------------------------------- | ---------- |
| A1  | LLM 模型選型 + 資源佔用       | 候選模型的中英文回答品質、prompt 格式。資源佔用需實測：SSD（模型檔大小）、VRAM（模型常駐）、GPU-Util（推論時算力佔用）、DRAM（推論時記憶體）、推論速度（tokens/sec）                   | §8.4, §8.5 |
| A2  | LLM 中英文能力             | 中文 chunks → 中文回答、英文 chunks → 中文回答、跨語言理解、長 context 多個 chunks 的理解能力                                                      | §5.2, §8.2 |
| A3  | Embedding 模型選型 + 資源佔用 | 候選模型的中英文語義匹配品質、維度。資源佔用需實測：SSD（模型檔大小）、VRAM（模型常駐）、GPU-Util（索引時持續 embedding 的算力佔用 vs 問答時單次 query 的算力佔用）、DRAM（批次處理時的記憶體佔用） | §8.4, §8.5 |
| A4  | Reranker 模型選型 + 資源佔用  | 候選模型的中英文重排品質。資源佔用需實測：SSD（模型檔大小）、VRAM（模型常駐）、GPU-Util（推論時算力佔用）、不同 chunks 數量（20/40/100）下的延遲和資源消耗                          | §8.4, §8.5 |
| A5  | vLLM 多 instance 部署    | 三個模型（LLM + Embedding + Reranker）需要分別跑不同的 vLLM instance，確認部署方式和 port 分配                                                 | §8.5       |
| A6  | LiteLLM 路由設定          | LiteLLM 設定 vLLM 本地 endpoint，確認 LLM / Embedding / Rerank 三種呼叫都能正確路由                                                     | §4.4       |
| A7  | Dify 模型供應商設定          | Dify 設定模型供應商指向 LiteLLM，確認知識庫建立時的切片 + embedding 能正常運作                                                                   | §4.4       |
| A8  | Embedding 換型影響        | 確認換 embedding model 是否需要全量 re-embedding                                                                                | §8.4       |


### B. Dify 部署相關


| #   | 項目                   | 測試內容                                                                                     | 相關章節 |
| --- | -------------------- | ---------------------------------------------------------------------------------------- | ---- |
| B1  | 共用 PostgreSQL        | Dify 指向 across 的 PostgreSQL（獨立 database `dify`），確認 Dify DB migration 能正常執行、不影響 across 資料 | §4.5 |
| B2  | 共用 Redis             | Dify 用 Redis db=2，確認 Celery broker 在非預設 db number 能正常運作、不與 across 的 key 衝突               | §4.5 |
| B3  | Dify 最小 container 驗證 | 逐一停用 sandbox、plugin_daemon、ssrf_proxy、worker_beat，測試知識庫上傳 + Retrieve API 是否仍正常           | §4.3 |
| ~~B4~~ | ~~Caddy 路由~~ | 短期不需要，across 後端走 Docker 內部網路直接打 Dify API | — |


### C. RAG Pipeline 相關


| #   | 項目                          | 測試內容                                                                         | 相關章節 |
| --- | --------------------------- | ---------------------------------------------------------------------------- | ---- |
| C1  | Dify Retrieve API 回應品質      | 上傳中英文文件後，Retrieve API 回傳的 chunks 是否相關、品質是否足夠                                 | §5.2 |
| C2  | 多 KB 並行呼叫                   | Promise.all 並行打多個 KB 的 Retrieve API，確認延遲、穩定性                                 | §5.3 |
| C3  | top_k 策略                    | 動態 top_k（總預算 / KB 數）vs 固定 top_k，哪個回答品質和延遲表現更好                                | §5.3 |
| C4  | 合併後一次 rerank                | 多 KB chunks 合併後一次 rerank 的品質和延遲，chunks 總量大時 reranker 表現                      | §5.3 |
| C5  | rerank 程式碼改造                | 現有 `rag-service.ts` 的 rerank 函式改 model name 指向 vLLM bge-reranker，確認 API 格式相容 | §6.1 |
| C6  | Worker 從 session 撈原始 prompt | 用 sessionId 從 DB 撈最後一條 user message，確認取得正確的原始 prompt                         | §D1  |
| C7  | Worker timeout              | KB 數量多、文件量大時，整條 pipeline（Retrieve + 合併 + rerank + LLM 回答）是否會超過 15 分鐘 timeout | §D   |
| C8  | rerank 是否應計入步數              | rerank 用 `fetch()` 不受步數限制保護，是否需改為 tool calling 方式實作 — 待討論                    | §D   |


### D. 雙 LLM 問題


| #   | 項目   | 測試內容                                    | 相關章節 |
| --- | ---- | --------------------------------------- | ---- |
| D1 | 速度影響 | Commander 2 次 + Worker 2 次 = 4 次 LLM 呼叫的實際延遲加總 | §9 |
| D2  | 資訊遺失 | Commander 轉發 Worker 回答時是否改寫、遺漏來源引用或專業術語 | §9   |
| D3  | 語意曲解 | Commander 對 Worker 回答的理解是否正確，特別是專業領域內容  | §9   |


### E. 上傳量控制


| #   | 項目                 | 測試內容                                                        | 相關章節 |
| --- | ------------------ | ----------------------------------------------------------- | ---- |
| E1  | 單次上傳檔案數量限制         | 前端和 API 層需限制一次上傳幾個檔案，避免大量檔案同時進入 Dify Worker embedding queue | §6.1 |
| E2  | 單檔大小上限             | 設定單檔大小上限，避免超大檔案卡住 Dify Worker                               | §6.1 |
| E3  | Dify Worker 併發處理量  | Celery concurrency 設定，控制同時幾個檔案在算 embedding，避免壓垮地端 vLLM      | §6.1 |
| E4  | vLLM embedding 吞吐量 | 地端 GPU 同時處理多少 embedding 請求的上限，超過時的排隊行為和延遲                   | §8.5 |


### F. 整合測試


| #   | 項目         | 測試內容                                                                       | 相關章節 |
| --- | ---------- | -------------------------------------------------------------------------- | ---- |
| F1  | 端到端 RAG 問答 | User 提問 → Commander → Worker → Dify Retrieve → rerank → LLM 回答 → 回傳前端，完整流程 | §5.2 |
| F2  | RBAC 正確性   | 不同角色只能查到自己被授權的 KB 內容，不會跨角色洩漏                                               | §4.1 |
| F3  | 文件上傳到可查詢   | MIS 上傳 md/docx/pdf/txt → Dify 切片 + embedding → 使用者提問能查到                    | §5.1 |


---

*本文件由 Claude Opus 4.6 基於 2026-05-31 與使用者的討論生成。*
*關鍵結論經 Gemini 2.5 Pro 和 OpenAI Codex (GPT-5.5) 交叉驗證。*