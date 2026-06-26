# Codex++「正在重新連線/5、stream disconnected」五人唯讀根因調查

- 調查者：td2（唯讀）
- 日期：2026-06-26
- gateway：210 生產（model.twister5.cf／210:4000）＝`across-litellm-stack`（litellm v1.85.1）
- 反映人：yiwen332、yanjie227、bonbon329、alex342、yizhen137
- 範圍：唯讀查 SpendLogs／VerificationToken／docker logs／caddy config。**未改任何設定**。

## 結論一句話
五人症狀（「正在重新連線/5」→「stream disconnected before completion: error sending request for url http://127.0.0.1:57321/v1/responses」）的**共同主因＝Cloudflare（model.twister5.cf）的 ~100s idle timeout 切斷大 context 的 /v1/responses 串流**。Codex++ 累積數萬~數十萬 token，reasoning model 思考靜默期超過 CF idle 上限 → CF 切斷 client↔210 連線 → Codex++ 報 stream disconnected 並重試（/5）；**而 litellm 端請求仍跑完、記成 200 success，完全不知情**（即既有記錄 cloudflare-100s-timeout-socket-closed 的 Codex 版）。另有兩個獨立疊加問題：yanjie227 model 名填錯、bonbon329 codex 上游 503。**此問題進行中、未解決**（基礎設施層，無人改 CF）。

---

## 五人症狀與資料對應
| 人 | 主要用法 | 關鍵數據 | 歸因 |
|---|---|---|---|
| yiwen332 | gpt-5.4/5.4-mini (responses) | success 為主；**有 1 筆 responses 142.9s** | CF timeout（重開 codex++＝清空 context 變快不撞）|
| alex342 | responses | 近 7 天**無記錄**（只 06-15 活動）；「時好時壞」| CF timeout（小 context 成功、大被切；部分請求 CF 就斷、未達 210）|
| yizhen137 | sonnet-4-6 **avg_pt=134,610**(/v1/messages) | 超大 context；「回答一半停」| CF timeout（大 context 串流被切）|
| yanjie227 | opus-4-6(avg_pt 99k)、gpt-5.4(responses) | + model 名 **`GPT5.4`** failure×3 | CF timeout **＋ model 名填錯**（雙因）|
| bonbon329 | **gpt-5.3-Codex**(responses) | failure 33/success 1，錯誤=**ServiceUnavailable** | 上游 MixRoute 503（codex 無管道，非 CF）|

## 共同主因：CF 100s idle timeout（高度坐實，排除法）

**證據鏈**
1. `/v1/responses`（Codex++）在 litellm 端 06-17~06-26 **全部 success、completion=0 為 0 筆**（litellm 視角無失敗、無截斷）。
2. **但 duration 長尾達 CF 上限以上**：max 06-22=167.5s、06-24=169.2s、06-25=140.9s；多筆 >100s。
3. 超大 context 是推手：responses 的 `gpt-5.5` max_prompt=**548,742**、`gpt-5.4` max=214,745 token；reasoning model 在如此大 context 下 thinking 靜默期極長（無 SSE data）。
4. **210 端無任何會在 100~169s 切斷的 timeout**：
   - litellm `request_timeout=300s`；
   - caddy 對 LLM 端點 `read_timeout 0`（不限）、SSE 自動 flush_interval -1；**且 caddy 反代清單根本沒有 litellm:4000**；
   - litellm:4000 = `0.0.0.0:4000` 直接對外。
   → 故 140-169s 的請求**不是 210 自己切的**；路徑上唯一會在 ~100s idle 切斷串流的是 **CF（model.twister5.cf）**。
5. 客戶端 error「stream disconnected before completion / error sending request」＝連線被中途切斷的典型；「正在重新連線/5」＝Codex++ 重試，重試帶同樣大 context → 再撞 → 5/5 用盡。
6. 「重開 codex++ 才好、重開 app 沒用」（yiwen332）＝重開 Codex++ 清空累積 context → 請求變小變快 → 不再撞 100s（重開 app 不清 Codex 的 conversation 狀態，故無效）。

※ 限制：server 端無法直接觀測 client 連線被 CF 切（litellm 記 200）。但「210 端所有 timeout 都 >169s」+「既有 CD 案已重現同機制」two 點，足以高度坐實 CF idle timeout。

## 個別疊加問題
- **yanjie227 — model 名填錯**：Codex++ 設定送 `GPT5.4`（大寫、無破折號），key.models 是 `gpt-5.4` → auth 層 `user_api_key_auth.py:1339` model-access 擋（與 skywalker codex 大小寫同類）。修＝改回 `gpt-5.4`。
- **bonbon329 — codex 上游 503**：`gpt-5.3-Codex`(responses) → `ServiceUnavailable`（MixRoute「無可用管道 distributor」，與 skywalker Issue 2′ 同源）。非 CF。需 MixRoute 端確認 codex distributor。

## 為何「上次回報沒被解決」（yizhen137）
CF idle timeout 是基礎設施層，至今無人調整 → 只要用「大 context + reasoning model + Codex++ /v1/responses」就週期性復發。這不是某次事件，是常態結構問題。

## 建議修法（不執行，待 Jeff 拍板）
| 優先 | 層級 | 措施 | 說明/風險 |
|---|---|---|---|
| ★高 | CF（根治） | 調高 model.twister5.cf 的 CF proxy/idle timeout，或對 /v1/responses 啟用 CF 對 SSE 友善設定 | 需 CF 方案支援（Enterprise 可調 100s）；或評估 Codex++ 直連 210:4000 繞過 CF（失去 CF 防護/TLS，需權衡）|
| ★高 | client | 引導用戶控制 Codex++ context（勿累積到數十萬 token）、降 reasoning effort、必要時用較快 model | 立即可緩解、零風險 |
| 中 | client | Codex++ 串流加 keep-alive/heartbeat，或縮短單請求；重試前先 compact | 取決於 Codex++ 是否可設定 |
| 中 | 設定 | yanjie227 改 model 名 `GPT5.4`→`gpt-5.4`；bonbon329 codex 503 找 MixRoute 確認 | 個別修正 |
| 低 | 監控 | 對「responses duration >90s」「同 key 連續重試」建告警，主動發現 CF 切斷 | litellm 記 success 故需用 duration 而非 status |

## 與前案關係
- skywalker007＝gateway 配置/相容 bug；kaiwei345＝opus 大 context 物理延遲；vic253＝06-16/17 已修復事件+流失。
- **本案＝進行中的 CF 100s idle timeout（基礎設施層），影響全體 Codex++ 重度/大 context 用戶**，與前案根因不同。bonbon 的 codex 503 與 yanjie 的 model 名錯，分別與 skywalker 案同源。

## 紅線聲明
全程唯讀，未改任何設定/vkey/model_list、未重啟容器、未碰 AIG 檔。CF 切斷為 server 端不可直接觀測、以排除法+已知機制高度坐實（已標注）。所有修法僅建議，需 Jeff 授權。報告無偽造資料。
