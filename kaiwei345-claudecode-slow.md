# kaiwei345 / Claude Code「思考速度偏慢」唯讀根因調查

- 調查者：td2（唯讀）
- 日期：2026-06-26
- gateway：210 生產（model.twister5.cf／210:4000）＝`across-litellm-stack`（litellm v1.85.1）
- 範圍：唯讀查 SpendLogs／VerificationToken。**未改任何 config／vkey／model_list／容器**。
- key：`kaiwei345`，token hash=`f261f6a5…c39877`，`blocked/tpm/rpm/max_parallel` 全 null（無限流），累計 spend≈$200.2。

## 結論一句話
**不是 bug、不是權限、不是 cache 沒打通** —— 是 Claude Code 用 **opus-4-8/4-7 + 累積到 10~24 萬 token 的超大 context**，造成回應時間長尾（p90 24~36s、p99 1~2 分鐘）。中位數其實只有 3~7s，體感慢來自長尾。屬大 context × reasoning model 的物理性延遲。

---

## 現象
kaiwei345 反映用 Claude Code 時「思考速度偏慢」。CC 走 Anthropic 協定 `/v1/messages?beta=true`，主力 model 為 opus。

## log 證據

**1. 用量與延遲百分位（success）**
| model | 筆數 | avg_prompt | p50 | p90 | p99 | max |
|---|---|---|---|---|---|---|
| `claude-opus-4-8` | 797 | **139,800** | 6.9s | 23.8s | 58.8s | 196.5s |
| `claude-opus-4-7` | 168 | **105,458** | 3.2s | 36.2s | 142.4s | 193.4s |
| `claude-haiku-4-5` | 28 | 4,732 | — | — | — | 2.1s（對照：鏈路本身很快）|

**2. context 越大越慢（opus，prompt 區間 vs avg_dur）**
```
  0-50k :  96 筆  5.9s
 50-100k: 331 筆  8.1s
100-150k: 230 筆 11.9s
150-200k: 139 筆 20.1s   ← 最慢區
200-250k:  63 筆 18.0s
250k+   : 106 筆 ~11s    ← 多為 cache hit、增量小
```

**3. prompt caching 有生效（response usage 實證）**
- 多數請求 `cache_read_input_tokens` 接近 prompt（如 237,944 → cache_read **235,400** ≈ 99% 命中）→ 12.4s。
- `cache_hit=False`(992 筆) 是 litellm 自身 response cache（本就未開，`supported_call_types:[]`），**與 Anthropic prompt cache 無關**，勿誤判。

**4. cache miss 偶發、非主因**
- >100k prompt 中：cache_hit(>50%) 516 筆 avg 14.4s；**cache_miss(<50%) 僅 22 筆（~4%）** avg 19.3s。
- 典型 miss 例：06-26 02:39 `cache_read 20,738 / cache_creation 206,674`（全量重建）→ **51.6s**。前一筆 02:10、間隔近 30 分鐘 > Anthropic ephemeral cache 5 分鐘 TTL → cache 過期重算。
- 但 cache_hit 的 max 仍達 193s → **長尾不全是 cache miss**，而是大 context＋opus thinking＋長 completion（最多 3,000+ tokens）＋偶發上游/排隊綜合。

**5. 並發非主因**
- kaiwei 活躍時段（06-26 01:45-02:45）每分鐘峰值 22 req 但僅 **2 個 key**，多為 kaiwei 自己連發；非多人擠壓。
- gateway 為**單 worker**（前次調查已確認），自己連發時請求間會排隊，放大長尾，但單筆瓶頸仍在上游處理 token。

## 根因判定（高度坐實，屬「正常但偏慢」）
1. **超大 context 是主因**：CC context 累積到 14~24 萬 token，即使 cache 99% 命中，載入大 KV cache + decode 仍需 10~40s；150k 區間平均已 20s。
2. **opus 是 reasoning model**：thinking + 長 completion 拉高耗時（completion 800~3,145 tokens）。
3. **偶發 cache miss（~4%）**：cache 5 分鐘 TTL 過期時全量重建 20 萬+ token → 50s+ 特慢。
4. **單 worker 連發排隊**：放大長尾。
非 gateway 配置錯誤、非權限 403、非 PII guardrail 主導。對照 haiku=0.5s，**鏈路健康**。

## 建議修法（不執行，待 Jeff 拍板）
| 優先 | 措施 | 預期效果 | 風險 |
|---|---|---|---|
| ★高 | 建議 kaiwei 在 CC 善用 `/compact`、`/clear` 控制 context，勿放任漲到 20 萬+ | 直接砍延遲（context 減半≈延遲減半） | 無，使用習慣 |
| ★高 | 非必要任務改用 `claude-sonnet-4-6`（CC 可切 model） | sonnet 比 opus 快很多 | 模型能力略降 |
| 中 | litellm 開多 worker | 改善連發/多人排隊長尾 | 需評估記憶體 |
| 中 | 確認 CC→litellm→MixRoute 是否透傳 extended cache TTL（`cache_control ttl:1h`） | 減少 ~4% cache miss 的全量重建 | 上游須支援，效益有限 |
| 低 | 評估 CC thinking budget 是否過高 | 縮短 thinking 耗時 | 影響推理品質 |

## 與 skywalker007 案的區別
skywalker 的問題是 gateway **配置/相容性 bug**（opus-4-5 配成 anthropic/gpt-5.5 必崩、codex 大小寫 403）。kaiwei 的問題**不是 bug**，是 opus + 巨大 context 的固有延遲。兩案根因互不相關。

## 紅線聲明
全程唯讀，未改任何設定/vkey/model_list、未重啟容器、未碰 AIG 檔。所有修法僅建議，需 Jeff 授權後執行。報告無偽造資料。
