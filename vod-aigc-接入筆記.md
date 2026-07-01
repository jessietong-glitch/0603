# 騰訊雲 VOD AIGC 生文能力 — 接入筆記

> 來源：《【對外】VOD AIGC 生文能力接入指南》
> 一個接入點調用多家 LLM（OpenAI / Gemini / Grok / Claude / MiniMax / Kimi / DeepSeek / GLM）
> 計費與原廠一致，每日結算出帳。

---

## 0. TL;DR（最快路徑）

```bash
# 1. 開通「雲點播 + 媒體處理」→ 自動產生 SubAppId
# 2. 呼叫 CreateAigcApiToken 拿到 TOKEN（無過期時間，不用每次請求都拿）
# 3. 拿 TOKEN 打端點：

curl https://text-aigc.vod-qcloud.com/v1/chat/completions \
  -H "Authorization: Bearer $VOD_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.4",
    "stream": false,
    "messages": [{"role":"user","content":"你好"}]
  }'
```

---

## 1. 端點總表

| 協議 | Base URL | 完整 URL | 適用模型 |
|---|---|---|---|
| OpenAI completions | `https://text-aigc.vod-qcloud.com/v1` | `…/v1/chat/completions` | gpt 系列（除 `gpt-5.4-pro`、`gpt-5.3-codex` 走 Responses）、gemini、gk、minimax、kimi、deepseek、GLM、claude |
| OpenAI 協議調 Claude | `https://text-aigc.vod-qcloud.com/openai/v1` | `…/openai/v1/chat/completions` | CD(Claude) 系列 |
| Anthropic 協議 | — | `https://text-aigc.vod-qcloud.com/v1/messages` | CD(Claude) 系列、minimax（官方推薦）|
| Responses 協議 | — | `https://text-aigc.vod-qcloud.com/v1/responses` | GPT 系列（含 `gpt-5.4-pro`、`gpt-5.3-codex`）|

請求方式一律 **POST**。

### 請求頭

| HTTP Header | 必需 | 說明 |
|---|---|---|
| `Authorization` | 是（若無 `x-api-key`）| `Bearer ${TOKEN}` |
| `x-api-key` | 是（若無 `Authorization`）| `${TOKEN}`，**Anthropic 協議用這種** |
| `X-Request-Id` | 否 | 可自帶 request id 方便追蹤 |

> Header 名稱不區分大小寫。Token 無過期時間。

---

## 2. 模型清單

| 廠商 | 可選值 | 備註 |
|---|---|---|
| OpenAI | `gpt-5.5`、`gpt-5.4-pro`、`gpt-5.4`、`gpt-5.4-mini`、`gpt-5.3-codex`、`gpt-5.3-chat`、`gpt-5.2-chat`、`gpt-5.2`、`gpt-5.1`、`gpt-5.1-chat`、`gpt-5`、`gpt-5-mini`、`gpt-5-nano`、`gpt-4.1`、`gpt-4o` | 上下文最大 1050k / 輸出 128k（依模型）；`5.4-pro`、`5.3-codex` 走 Responses 協議 |
| Gemini | `gemini-3.1-pro-preview`、`gemini-3.1-flash-lite-preview`、`gemini-3-flash-preview`、`gemini-2.5-pro`、`gemini-2.5-flash` | 支援文字/程式碼/圖/音訊/視訊；輸入 1,048,576 / 輸出 65,536 |
| GK (Grok) | `gk-4-1-fast-reasoning` | ⚠️ **僅國際站**；上下文 2M |
| CD (Claude) | `cd-opus-4.7`、`cd-sonnet-4.6`、`cd-opus-4.6`、`cd-opus-4.5`、`cd-haiku-4.5` | ⚠️ **僅國際站**；建議走 Anthropic 協議 |

> `gpt-5`(chat) 官方 4/15 已下架、`gemini-3-pro-preview` 官方 3/9 已下線。資源緊缺或超過 200w TPM 的模型需**提前 3 天申請**。

---

## 3. 各協議調用範例

### 3.1 OpenAI Completions

**curl**
```bash
curl https://text-aigc.vod-qcloud.com/v1/chat/completions \
  -H "Authorization: Bearer $VOD_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.4",
    "stream": false,
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "用一句話介紹騰訊雲點播"}
    ],
    "temperature": 0.7,
    "max_tokens": 1024
  }'
```

**Python（OpenAI SDK）**
```python
from openai import OpenAI

client = OpenAI(
    api_key="VOD_TOKEN",
    base_url="https://text-aigc.vod-qcloud.com/v1",
)

resp = client.chat.completions.create(
    model="gpt-5.4",
    messages=[{"role": "user", "content": "你好"}],
    stream=False,
)
print(resp.choices[0].message.content)
```

**Node（openai SDK）**
```javascript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.VOD_TOKEN,
  baseURL: "https://text-aigc.vod-qcloud.com/v1",
});

const resp = await client.chat.completions.create({
  model: "gpt-5.4",
  messages: [{ role: "user", content: "你好" }],
});
console.log(resp.choices[0].message.content);
```

### 3.2 用 OpenAI 協議調 Claude（換 base URL）
```python
client = OpenAI(
    api_key="VOD_TOKEN",
    base_url="https://text-aigc.vod-qcloud.com/openai/v1",  # 注意多了 /openai
)
resp = client.chat.completions.create(
    model="cd-sonnet-4.6",
    messages=[{"role": "user", "content": "你好"}],
)
```

### 3.3 Anthropic 協議（原生調 Claude / minimax）

**curl**（注意用 `x-api-key`）
```bash
curl https://text-aigc.vod-qcloud.com/v1/messages \
  -H "x-api-key: $VOD_TOKEN" \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "cd-opus-4.7",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "你好"}]
  }'
```

**Python（anthropic SDK）**
```python
from anthropic import Anthropic

client = Anthropic(
    api_key="VOD_TOKEN",
    base_url="https://text-aigc.vod-qcloud.com",  # SDK 會自動補 /v1/messages
)
msg = client.messages.create(
    model="cd-opus-4.7",
    max_tokens=1024,
    messages=[{"role": "user", "content": "你好"}],
)
print(msg.content[0].text)
```

### 3.4 Responses 協議（GPT 系列 / codex / 5.4-pro）
```bash
curl https://text-aigc.vod-qcloud.com/v1/responses \
  -H "Authorization: Bearer $VOD_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.3-codex",
    "input": "寫一個 Python quicksort"
  }'
```

---

## 4. 常用請求參數（OpenAI Completions）

| 參數 | 型別 | 必填 | 說明 |
|---|---|---|---|
| `model` | string | 是 | 見模型清單 |
| `messages` | list | 是 | 對話上下文 |
| `stream` | bool | 是 | 是否流式 |
| `thinking_enabled` | bool | 否 | 開啟深度思考(CoT)，會產生 `reasoning_tokens`；`gpt-5.1/5.2/4o` 不支援、`gemini-3-*` 特殊 |
| `temperature` | float | 否 | 0~2，預設約 0.7 |
| `max_tokens` | int | 否 | 單次最大 token（含輸入+輸出）|
| `reasoning_effort` | string | 否 | `none/minimal/low/medium/high/xhigh`；與 `thinking_enabled` 同設時以此為準 |
| `tools` / `tool_choice` | array/obj | 否 | Function calling（3.11 起支援；gemini 只支援 function）|
| `response_format` | obj | 否 | 結構化輸出（text / json_schema / json_object）|

**多模態 content parts**：`text`、`image_url`（含 data URL，≤70MB）、`input_audio`（base64，mp3/wav）、`file`（file_url，≤70MB）。

---

## 5. 錯誤碼與限速

| HTTP | 意義 |
|---|---|
| 200 | 成功 |
| 400 | 請求參數錯誤 |
| 401 | 認證失敗（token 錯）|
| 403 | 權限不足 / 已停服（可能欠費）|
| 404 | 模型或端點不存在（model 名不支援 / path 錯）|
| 429 | 速率限制 |
| 5xx | 伺服器 / 上游錯誤 |

**預設限速**：RPM 30、TPM 30 萬 tokens（測試期 TPM 可能僅 10）。
需調整時提供 **AppId + 模型 + 需要的 RPM/TPM** 給對接窗口。

---

## 6. 接入 LiteLLM（對應你的部署場景）

VOD 是 OpenAI 相容端點，可直接當 `openai/` provider 掛進 LiteLLM。

```yaml
# litellm config.yaml
model_list:
  # --- OpenAI / Gemini / Grok 等走 completions ---
  - model_name: vod-gpt-5.4
    litellm_params:
      model: openai/gpt-5.4
      api_base: https://text-aigc.vod-qcloud.com/v1
      api_key: os.environ/VOD_TOKEN

  - model_name: vod-gemini-3-flash
    litellm_params:
      model: openai/gemini-3-flash-preview
      api_base: https://text-aigc.vod-qcloud.com/v1
      api_key: os.environ/VOD_TOKEN

  # --- Claude 走 Anthropic 原生協議 ---
  - model_name: vod-claude-opus-4.7
    litellm_params:
      model: anthropic/cd-opus-4.7
      api_base: https://text-aigc.vod-qcloud.com
      api_key: os.environ/VOD_TOKEN

  # --- 或用 OpenAI 協議調 Claude（換 /openai/v1）---
  - model_name: vod-claude-sonnet-4.6-oai
    litellm_params:
      model: openai/cd-sonnet-4.6
      api_base: https://text-aigc.vod-qcloud.com/openai/v1
      api_key: os.environ/VOD_TOKEN
```

```bash
export VOD_TOKEN="<CreateAigcApiToken 拿到的 token>"
litellm --config config.yaml

# 驗證
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"vod-gpt-5.4","messages":[{"role":"user","content":"ping"}]}'
```

> 注意：`gpt-5.4-pro`、`gpt-5.3-codex` 只走 Responses 協議，LiteLLM 需用支援 responses 的路由；一般 chat 場景用其他 gpt 型號即可。

---

## 7. Token 管理接口（騰訊雲 API）

| 動作 | 接口 | 說明 |
|---|---|---|
| 建立 | `CreateAigcApiToken` | 建 token 需帶 `SubAppId` |
| 刪除 | `DeleteAigcApiToken` | |
| 查詢 | `DescribeAigcApiTokens` | 列出 token |

- 可在騰訊雲 **API Explorer** 直接產生，或參考其產的程式碼。
- **國際站與國內站 APIKey 不通用**；Grok、Claude 等只能用國際站。
- 子帳號調用需主帳號授權（可全部 SubAppId 或指定部分）。

**平台入口**
- 國內站：`https://console.cloud.tencent.com/vod`
- 國際站：`https://www.tencentcloud.com/zh/products/vod`

---

## 8. 用量查詢

| 方式 | 位置 |
|---|---|
| 接口 | `DescribeAigcUsageData`（國內站文件 266/126446；國際站 266/78365）|
| 可視化 | 雲點播 → 用量統計 → AIGC |
| 成本分析 | 費用 → 成本分析（resourceId 維度）/ 資源帳單 / 明細帳單 |

> Q：明細帳單用量和價格不對？
> A：用量看配置描述，單價看官方《VOD AIGC 價格指南》。

---

## 9. 重點提醒（踩雷點）

1. **協議選錯 = 404**：codex / 5.4-pro 必走 Responses；Claude 建議走 Anthropic。
2. **Anthropic 協議用 `x-api-key`，不是 `Authorization`**。
3. **國際站 / 國內站 token 不通用**，Grok、Claude 僅國際站。
4. **測試期 TPM 很低（可能 10）**，正式接入再申請調高。
5. Token 沒有過期時間，妥善保存、勿硬編進 repo。
