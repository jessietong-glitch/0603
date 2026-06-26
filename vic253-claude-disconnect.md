# vic253 / Claude「斷線、溝通結果作廢」唯讀根因調查

- 調查者：td2（唯讀）
- 日期：2026-06-26
- gateway：210 生產（model.twister5.cf／210:4000）＝`across-litellm-stack`（litellm v1.85.1）
- 範圍：唯讀查 SpendLogs／VerificationToken／docker logs。**未改任何設定**。
- key：`vic253`，token hash=`8febba01…ecaa6b`，created 06-12，**max_budget=null**（key 自身無預算上限），spend≈$21.9，無 rpm/tpm/parallel 限流。

## 結論一句話
vic 的「斷線、之前溝通作廢」**不是網路、不是 CF timeout、不是回應慢、也不是 vic 個人超支**，而是 **2026-06-16 與 06-17 連續兩天、兩個不同的系統層事件**把他的請求在送達模型前就擋下，client 該輪對話作廢。**兩個事件目前都已結束、系統現況正常**，但 vic 在 06-17 那波失敗後即流失、轉用 copilot/codex，至今（9 天）未回——他不知道已經好了。**且兩事件都是 tonwa 群體受害，非 vic 個案。**

---

## 現象
vic 反映 Claude 有時斷線，斷線前的溝通結果全部作廢、浪費時間，現多用 copilot 與自己的 codex。

## 排除的假設（附證據）
- **非 CF 100s idle timeout**：vic 請求 >100s 僅 1 筆（425 筆都在 0-30s），延遲不是問題。
- **非回應慢**：opus-4-6 avg_dur 3.9s／p90 6.3s；sonnet-4-6 avg 3.9s／p90 7.6s，都很快。
- **非 vic 個人超支**：vic key `max_budget=null`，spend 僅 $21.9。

## vic 失敗的精確構成（坐實）
vic 全部活動只在 06-16 01:49 ~ 06-17 06:34（318 success + **113 failure**，均 pt=0/ct=0/dur=0 空殼＝送達模型前即失敗），按日與錯誤分類：

| 日期 | budget exceeded | get_key_object 401 | 合計 |
|---|---|---|---|
| 06-16 | **20** | 0（4 其他）| 24 |
| 06-17 | 0 | **89**（86 sonnet＋3 haiku）| 89 |

→ **兩天是兩個不同根因。**

## 事件一：06-16 budget 誤判（已修復）
- 錯誤：auth 層 `ProxyException: Budget has been exceeded`。
- 證據：docker logs 14 天「Budget has been exceeded」13,456 次，**全集中在 06-12~06-16**，近 10 天=0。
- 根因：litellm v1.85 cross-pod budget 在未接 Redis 時 in-memory 虛漲撞系統層上限（即既有事件記錄 litellm-global-budget-429-incident）。**非 vic 個人 budget**（他 max_budget=null）。
- 修復：2026-06-16 06:14Z 將真人 key max_budget 全設 NULL（+ 接 Redis），budget 429 歸零。

## 事件二：06-17 真實 key 被誤拒 401（已結束，係事件一修復的副作用）
- 錯誤：`auth_checks.py:2385 get_key_object → raise ProxyException, error_code:"401"`（經 `log_db_metrics.py` wrapper），traceback 在 `user_api_key_auth.py`。
- **底層非 DB timeout/連線/pool/redis**（逐一比對 metadata 皆 0 命中），是 get_key_object 查不到有效 key 而拋 401。
- 關鍵區分（get_key_object 401 真實 key vs unknown 無效 key，逐日）：
  ```
  06-12~16:  真實key=0    （全是 unknown 無效 key＝正常拒絕）
  06-17   :  真實key=1661  unknown=672   ← 唯一一天真實 key 被大規模誤拒
  06-18起 :  真實key=0    （06-22/24/26 的 203/176/34 全是 unknown 無效 key，非故障）
  ```
- 判定：**只有 06-17 一天**，真實 tonwa key 被 get_key_object 誤拒 401（1,661 筆）。時間緊接事件一修復（06-16 06:14Z 大量 key UPDATE + 重啟），**高度疑似 key cache 在批量 key 操作後暫時不一致**，導致真實 key 查不到→401。06-18 起歸零、徹底結束。
- ⚠️ **澄清易誤判點**：06-22/24/26 仍有 get_key_object 401，但**全部是 unknown 無效 key（外部掃描/過期/設錯的 client），屬正常拒絕，不是系統故障**。今天系統對真實 tonwa key 正常。

## vic 流失時間線（鐵證）
- 06-16：被 budget 擋 20 筆。
- 06-17：被 key-401 擋 89 筆（06-17 06:33-06:34 連續 sonnet 失敗）。
- 06-17 06:34 最後一筆失敗後，**至今 9 天無任何請求** → 完全吻合「改用 copilot/codex」。
- 諷刺點：06-18 起兩事件皆消、系統正常，但 vic 已走、不知已修復。

## 群體影響（非 vic 個案）
- 事件一（06-15 budget）受害者：dio075(1392)、jack130(1047)、ray084(1021)、leo301、vincent357…
- 事件二（06-17 key-401）真實 key 受害者：**leo301(461)、yanjie227(335)、yuwen034(198)、vincent357(127)、yuting352(108)、vic253(89)、yizhen137(89)、tom088(82)…** 與事件一高度重疊——同一批人連續被兩波打。

## 建議行動（不執行，待 Jeff 拍板）
技術問題**已解決**，重點是「挽回 + 防複發 + 監控」：

| 優先 | 行動 | 說明 |
|---|---|---|
| ★高 | **主動挽回 vic253 等流失者** | 告知「06-16/17 的問題已修復、現已穩定」，邀請回來。vic 已轉競品需主動 outreach。 |
| ★高 | **盤點 06-15/06-17 受害者回流狀況** | leo301/yanjie227/yuwen034/vincent357/dio075/jack130… 逐一確認誰像 vic 一樣斷了沒回。 |
| 中 | **防複發** | 批量 key 操作（UPDATE/restart）後 key cache 易暫時不一致 → 應 warm cache 或滾動重啟，避免再現 06-17 的真實 key 誤拒。 |
| 中 | **加監控告警** | 對「真實 key 的 401／budget exceeded／全體失敗率突升」設告警（要能區分真實 key vs unknown），別再讓客戶踩雷後才從回報得知。 |

## 與前兩案的關係
- skywalker007＝gateway 配置/相容性 bug（opus-4-5、codex 大小寫）。
- kaiwei345＝opus 大 context 物理延遲（非 bug）。
- vic253＝**兩個已結束的系統事件（budget 誤判 + key-cache 401）造成的客戶流失**。
三案根因互不相同；vic 案的價值在於揭露 06-16/06-17 連續事件造成 tonwa 群體流失，需業務面挽回 + 流程防複發。

## 紅線聲明
全程唯讀，未改任何設定/vkey/model_list、未重啟容器、未碰 AIG 檔。所有行動僅建議，需 Jeff 授權。報告無偽造資料；不確定處（06-17 key cache 不一致為「高度疑似」）已標注。
