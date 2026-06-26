# skywalker007 / Codex++ 五問題 唯讀根因調查

- 調查者：td2（唯讀）
- 日期：2026-06-26
- 目標 gateway：210 生產（model.twister5.cf／210.208.232.139:4000）＝容器 `across-litellm-stack`（litellm v1.85.1）
- 授權範圍：唯讀查 log/設定。**未改任何 config／vkey／model_list／容器**。
- 證據來源：`docker inspect`、host 端 `litellm_config.yaml`(ro)、Postgres `across-litellm-stack-db`（SpendLogs / VerificationToken / ProxyModelTable 唯讀）、`docker logs`、litellm `/v1/model/info`（master key 唯讀 GET）。

---

## 0. 環境定位（查證後事實，非假設）

- **skywalker007 的 vkey**：key_alias=`skywalker007`，token hash=`2b7cf581…b39f9`，`models` 陣列 15 個（與 403 訊息清單一致），`blocked/tpm/rpm/max_parallel` 全 null（**此 key 無任何限流**）。走 210 生產 gateway。
- **model_list 來源**：config.yaml 只有 `qwen3.6` 一個；其餘全部靠 `STORE_MODEL_IN_DB=True` 存在 DB（UI 新增）。
- **底層路由（/v1/model/info 實際運行值）**：

  | gateway model_name | 底層 model 字串 | provider | 備註 |
  |---|---|---|---|
  | `gpt-5.5` | `gpt-5.5` | openai | 正常（MixRoute /v1/responses） |
  | `gpt-5.3-Codex` | `gpt-5.3-Codex` | openai | 僅大寫一筆 |
  | **`claude-opus-4-5`** | **`anthropic/gpt-5.5`** | anthropic | ★配置異常，見 Issue 3 |
  | `claude-opus-4-7` | `anthropic/claude-opus-4-7` | — | 正常對照 |
  | `claude-sonnet-4-6` | `anthropic/claude-sonnet-4-6` | — | 正常對照 |
  | `claude-opus-4-8` | `claude-opus-4-8` (openai) ＋ `anthropic/claude-opus-4-8` (anthropic) | 兩筆 | 重複，附帶風險 |
  | `claude-haiku-4-4` | `hosted_vllm/Qwen/Qwen3.6-35B-A3B-FP8` | hosted_vllm | 已知掛 Qwen |

- **router/settings**：`routing_strategy: least-busy`、`num_retries: 2`、`allowed_fails: 1`、`cooldown_time: 3600`(1h)、`enable_pre_call_checks: true`；`drop_params: true` + `additional_drop_params: ["reasoning_effort"]`；`request_timeout: 300`；Redis cache 已接（namespace litellm-stack，`supported_call_types: []`＝僅用於 spend/限流不快取回應）；Presidio PII pre_call guardrail（遮罩 TW_NATIONAL_ID／IPV4／CREDIT_CARD）；**未設任何 fallbacks**。
- **worker 數**：startup log 僅 `Started server process [1]` + 單一 Uvicorn，**無多 worker（無 gunicorn）= 單 worker**。
- **Codex++ 請求特徵**：payload 帶 `stream:true` + `tools:[shell_command…]`（function calling / PowerShell agent），prompt 動輒 2萬~11萬 token。

### skywalker007 同一 session 時間線（06-22 上午，與五問題完全對應）
```
06:33-06:34  gpt-5.5 (/v1/responses)          → completion_tokens=5     = 問題1
06:35        gpt-5.3-codex (小寫)             → request body 空 = 被擋   = 問題2
06:39-06:44  claude-opus-4-5(記為anthropic/gpt-5.5) → 13連敗 system bug  = 問題3
06:44-06:46  gpt-5.3-Codex (大寫)             → 上游MixRoute無管道 503   = 問題2(另一路徑)
```

---

## 問題 2 ＋ 其他(1)「兩個 Codex」【同源，合併】

**現象**：Codex++ 報 403「key not allowed… can only access models=[…'gpt-5.3-Codex'…]. Tried to access **gpt-5.3-codex**」；且選單出現兩個 GPT-5.3-Codex。

**log 證據**
- SpendLogs（skywalker007）：`gpt-5.3-codex`(小寫) failure×24、`gpt-5.3-Codex`(大寫) failure×3。
- 小寫失敗筆：`proxy_server_request = {}`（空）→ 請求在 LLM call 之前就被擋（auth/路由層拒絕的特徵）。
- 大寫失敗筆：`proxy_server_request` 完整含 `"model":"gpt-5.3-Codex"`+tools → 有進到上游，docker logs 對應 `ServiceUnavailableError … "分組 default 下模型 gpt-5.3-Codex 無可用管道（distributor）" type:mixroute_error`（168h 計 20 次，**近 24h＝0**）。
- DB `ProxyModelTable` 與 `/v1/model/info`：**只有大寫 `gpt-5.3-Codex` 一筆**，無小寫版。
- key.models 陣列字面值＝`gpt-5.3-Codex`（大寫）。
- docker logs 168h 內「not allowed to access」字串＝0 筆（該 403 發生在 168h 之前，或由客戶端/auth 早退路徑產生而未落 stdout；但小寫失敗的空 body 與 403 文字一致）。

**根因判定**
- 403：**已坐實＝大小寫不一致**。gateway 註冊與 key 允許清單都是 `gpt-5.3-Codex`（大寫 C），Codex 生態慣用小寫 `gpt-5.3-codex`，litellm 的 model 比對大小寫敏感 → 小寫請求不在允許清單 → 403。
- 上游 503「無可用管道」：**已坐實但近期已緩解**（24h=0），屬 MixRoute 後台該型號 distributor 狀態，非 litellm 問題。
- 「選單兩個」：**高度疑似客戶端兩來源**——gateway 端僅一筆（大寫），故兩個來自 Codex++ 自身（一個取自 gateway `/v1/models` 的大寫、一個取自 Codex 預設/設定的小寫）。gateway 端目前無 codex 重複（但 `claude-haiku-4-5`／`claude-opus-4-8`／`gpt-5.4-mini`／`vercel…gemini-2.5-flash` 在 DB 各有兩筆，未來會造成這些 model 的選單重複）。

**建議修法（不執行，待拍板）**
1.（推薦）統一成小寫 `gpt-5.3-codex`：DB/UI 改 model_name 為小寫，並同步全部相關 key 的 `models` 清單字串為小寫。一次解決 403＋選單命名。
2. 或最小改動：在每把 key 的 `models` 同時加入大小寫兩個字串（影響 38 把 key，較髒）。
3. 上游 503 需 MixRoute 後台確認 `gpt-5.3-Codex` distributor 是否穩定（已 24h 無事，先觀察）。

---

## 問題 3「Claude-Opus-4.5 思考極慢」【真相＝必崩，非慢】

**現象**：使用者覺得 opus-4.5 思考極慢、轉圈不回。

**log 證據**
- SpendLogs：`claude-opus-4-5` 全域 14 天 **47 筆全 failure，completion_tokens=0、TTFT=0**（從未產出 token）。
- docker logs（06-22 06:43 等）：`litellm.InternalServerError: AnthropicException - AsyncCompletions.create() got an unexpected keyword argument 'system'`，位置 `litellm/proxy/anthropic_endpoints/endpoints.py:53`，`Received Model Group=claude-opus-4-5`（13 筆，即 skywalker 06-22 那批，SpendLogs 記為 `anthropic/gpt-5.5`）。
- **`/v1/model/info`：`claude-opus-4-5` 的底層 model 字串＝`anthropic/gpt-5.5`、provider=anthropic**（對照正常的 `claude-opus-4-7→anthropic/claude-opus-4-7`）。

**根因判定：已坐實＝model 配置錯誤。**
`claude-opus-4-5` 的 underlying 被填成 `anthropic/gpt-5.5`（而非 `anthropic/claude-opus-4-5`）。當 Codex++ 以 Anthropic 協定（/v1/messages，含 top-level `system`）打它，litellm 的 anthropic 端點處理鏈最終以 OpenAI 的 `AsyncCompletions.create()` 呼叫並帶入 `system` 參數 → TypeError → InternalServerError，故每次必崩、零輸出，客戶端表現為「一直思考不回」。
※附帶事實：即使少數 success，opus-4-5 實際打的也是 gpt-5.5（掛名 Opus 實為 gpt-5.5）。**請 Jeff/Jimmy 確認此配置是「打字／複製錯誤」還是刻意掛名**——兩種解讀的修法不同，td2 不臆測。

**建議修法（不執行，待拍板）**
1.（推薦，若為配置錯誤）把 `claude-opus-4-5` 的底層 model 改為正確的 `anthropic/claude-opus-4-5`（比照 4-6/4-7/4-8 的 anthropic 寫法），即同時解決崩潰與掛名。
2. 若刻意掛 gpt-5.5：至少要讓它能服務 /v1/messages（provider 與協定一致），否則 Codex++ 永遠崩。

---

## 問題 1「GPT-5.5 讀檔中斷：回『要讀取檔案』就沒下文」

**現象**：請 gpt-5.5 讀 README 總結 3 句，回「要讀取檔案」後沒下文；其他模型正常。

**log 證據**
- SpendLogs（skywalker007，model=`gpt-5.5`，走 openai `/v1/responses`）：success 但 **completion_tokens=5**（06-22 06:33，prompt=77229）、=55、=122；其他歷史筆亦見 ct=5/10。對照 `gpt-5.4` success×163，completion 動輒數百~1700 正常。
- 該 ct=5 回應內容：`"status":"completed"`、`"incomplete_details":null`、`"reasoning":{"effort":"medium"…}`，output item 型別為 `message`（**非 `function_call`**），response id 以 `resp_…`（Responses API）。
- ＝模型「正常結束」（非截斷、非超 token、上游無報錯），只是只回了極短一句文字、**沒有發出工具呼叫**。Codex++ 拿到「說要讀檔卻沒給 tool_call」的回應 → agent loop 無工具可執行也無完整答案 → 卡住「沒下文」。

**根因判定：高度疑似＝gpt-5.5 在 Codex(/v1/responses + tools)場景的工具呼叫行為異常**——只輸出意圖文字、不產生 function_call，而 gpt-5.4 在相同客戶端可正常發 tool_call 並完成。litellm 端 `additional_drop_params:["reasoning_effort"]`＋`drop_params:true` 可能影響 reasoning model 行為（屬貢獻因素、非已證主因）。屬模型／上游相容性，非 key/權限問題。

**建議修法（不執行，待拍板）**
1.（短期）建議 tonwa 同仁在 Codex++ 跑 agentic（讀檔/工具）任務改用 `gpt-5.4`（實測穩定）；gpt-5.5 暫作非工具對話。
2.（待驗證）檢視 litellm 對 gpt-5.5 /v1/responses 的參數透傳：不要 drop `reasoning_effort`、確認 `tools`/`tool_choice` 完整透傳到 MixRoute；之後重打同一 Codex 請求比對。
3. 需要時可向 MixRoute 反映 gpt-5.5 在 Responses+function-calling 的回應退化。

---

## 其他(2)「多人使用時回應有點慢」

**log 證據**
- **單 worker**（uvicorn process[1]，無多 worker）。
- 168h：`timed out/timeout` 命中 1677、`429` 命中 621、cooldown/over-limit 類 4。
- 成功請求多數走 `https://api.mixroute.ai/...`（429 多源自上游 MixRoute 限速）。
- Codex++ 請求 prompt 巨大（2萬~11萬 token），每筆都先過 Presidio pre_call PII 掃描（掃整個 prompt）。
- `cooldown_time=3600`＋`allowed_fails=1`：任一 model 一次失敗即冷卻 1 小時，期間該 model 不可用，會放大「卡頓」感。

**根因判定：結構性＝高度疑似。** 單 worker × 大 prompt × reasoning 長耗時 × 每請求 Presidio 掃描，在並發時排隊明顯；疊加上游 429／過長 cooldown。與既有記憶「litellm 多 worker、per-key 限流屬待辦」一致。

**建議修法（不執行，待拍板）**
1. litellm 開多 worker（`--num_workers`／或 gunicorn workers，需評估記憶體）。
2. 檢視 Presidio pre_call 對大 prompt 的延遲（必要時改 async/縮小掃描範圍）。
3. `cooldown_time` 由 3600 調短（如 120-300）、重新評估 `allowed_fails`。
4. 視情況加 per-key rpm/tpm 限流（目前 skywalker007 等 key 全無限流）。

---

## 總表

| # | 問題 | 根因 | 信心 | 推薦修法 |
|---|---|---|---|---|
| 2 | codex 403（小寫） | model 名大小寫不一致（gateway/key=大寫，客戶端送小寫），litellm 比對大小寫敏感 | 已坐實 | 統一小寫 + 同步 key.models |
| 2′ | codex 大寫 503 | MixRoute 上游無 distributor（近24h已緩解） | 已坐實 | 觀察/MixRoute 端 |
| 1bonus | 選單兩個 codex | 客戶端兩來源（gateway 僅一筆） | 高度疑似 | 統一命名後消失 |
| 3 | opus-4-5「慢」 | 底層配成 `anthropic/gpt-5.5`，走/v1/messages 撞 `AsyncCompletions … 'system'` → 必崩 | 已坐實 | 改回 `anthropic/claude-opus-4-5`（先確認是否刻意） |
| 1 | gpt-5.5 讀檔中斷 | gpt-5.5 在 /v1/responses+tools 只回短訊息、不發 function_call → Codex loop 卡住 | 高度疑似 | 暫用 gpt-5.4 跑 agent；查參數透傳 |
| 其他2 | 多人慢 | 單 worker＋大prompt＋Presidio＋上游429＋長cooldown | 結構性/高度疑似 | 多 worker、調 cooldown、Presidio、限流 |

## 紅線聲明
全程唯讀，未改 litellm config／未改任何 vkey／未動 model_list／未重啟容器；未碰 td 的 AIG 檔。所有修法僅為建議，需 Jeff 授權後才執行。報告無偽造資料；取不到者已標注。
