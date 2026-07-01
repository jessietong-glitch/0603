# Hermes 意圖路由層（task-router）— 設計、原理與接入說明

> 建立日期：2026-06-30　主機：DGX Spark（aitopatom-b624）
> 目的：讓使用者「一句話 + 素材」交派任務，系統自動派給正確的流程/模型可靠執行，
> 不必每次貼長提示或手動 `/model` 切換。

---

## 0. 一句話總結

在 Hermes 收到使用者訊息、交給 AI agent **之前**，插入一層「**意圖路由**」：
偵測這句話想做什麼 → 命中配方就把訊息**自動展開成完整指令**並**綁定對的模型** → agent 收到的是「已經講清楚的任務」→ 可靠執行。

---

## 1. 為什麼需要這層（問題背景）

### 原本的困境
Hermes 的 **skill 機制是「模型自己選」**：把 skill 的描述放進 context，由 LLM 自行決定要不要讀、要不要照做。

- 在雲端強模型（如 Claude）上：模型夠聰明，能可靠地「自己選對 skill + 照做」→ 一句話就行。
- 在**地端 30B 模型**（qwen3-coder / nemotron3）上：這個「自我編排」能力較弱 → 常常：
  - 不去讀對的 skill、自由發揮（改用陽春 python-pptx、產空白頁、瞎掰假資料）
  - 把文件當成空白模板、反問使用者要資料
  - 完全忽略既有的製作流程

### 核心洞察
> **不要靠（不夠強的）模型自己即興編排，把「該用什麼流程」的判斷用程式寫死，變成確定性的路由。**

這跟 Claude Code 的「workflow」是同一套哲學的**前門（dispatch）那一層**：
- **意圖路由層** = 前門：判斷要做什麼 → 派給對的流程（本文件主題）
- **執行流程（pipeline/skills）** = 引擎：把任務按步驟可靠跑完（先前已建好）

---

## 2. 接入點：在 Hermes 的哪裡新增

### 2-1. 使用的官方擴充點：`pre_gateway_dispatch` hook

Hermes gateway 在處理每一則 inbound 訊息時，**在 agent 執行前**會觸發 plugin hook：

- **程式位置**：`/usr/local/lib/hermes-agent/gateway/run.py`（約第 6381 行，函式 `_handle_message`）
- **觸發方式**：`hermes_cli/plugins.py` 的 `invoke_hook("pre_gateway_dispatch", event=..., gateway=..., session_store=...)`
- **此 hook 在 `VALID_HOOKS` 清單內**（`hermes_cli/plugins.py`）
- **plugin 可回傳的動作**：
  | 回傳 | 效果 |
  |------|------|
  | `{"action": "rewrite", "text": "新內容"}` | **替換** event.text（agent 收到改寫後的訊息）← 我們用這個 |
  | `{"action": "skip", "reason": ...}` | 丟棄訊息（plugin 自行處理）|
  | `{"action": "allow"}` / `None` | 正常放行 |

> 關鍵：這是 Hermes **官方支援的 plugin hook**，所以我們**完全不需要修改核心程式碼**，
> `hermes update` 也不會覆蓋我們的東西。

### 2-2. 為什麼用「使用者 plugin」而非改核心

Hermes plugin 載入順序（`hermes_cli/plugins.py`）：
1. Bundled plugins：`/usr/local/lib/hermes-agent/plugins/`（核心、會被更新覆蓋）
2. **User plugins：`~/.hermes/plugins/<name>/`** ← 我們放這裡（你的目錄、更新不覆蓋）
3. Project plugins：`./.hermes/plugins/<name>/`

→ 把路由做成**使用者 plugin**，是**最乾淨、最安全、不被更新覆蓋**的接入方式。

---

## 3. 我們新增的元件

### 3-1. task-router plugin（路由引擎，建一次）

位置：`~/.hermes/plugins/task-router/`

```
task-router/
├── plugin.yaml      # 宣告 plugin、註冊 pre_gateway_dispatch hook
├── __init__.py      # 路由邏輯（偵測意圖 → 改寫訊息 → 切模型）
└── recipes.yaml     # 配方庫（每種任務一筆；擴充任務改這裡，不改程式）
```

**plugin.yaml**（宣告 hook）：
```yaml
name: task-router
hooks:
  - pre_gateway_dispatch
```

**__init__.py 的核心邏輯**：
```python
def register(ctx):
    ctx.register_hook("pre_gateway_dispatch", _on_pre_gateway_dispatch)

def _on_pre_gateway_dispatch(event=None, gateway=None, session_store=None, **_):
    try:
        text = getattr(event, "text", "")
        recipe = _match_recipe(text)          # 比對 recipes.yaml 觸發詞
        if recipe is None:
            return None                        # 沒命中 → 正常聊天
        _maybe_switch_model(gateway, event, recipe["model"])   # 自動切模型
        new_text = recipe["expand"] + "\n\n【使用者原始需求】\n" + text
        return {"action": "rewrite", "text": new_text}         # 改寫訊息
    except Exception:
        return None                            # 任何錯誤 → 放行(絕不弄壞訊息流)
```

**安全設計（重點）**：整個 callback 包在 `try/except`，**任何例外一律回 `None`（正常放行）**，
所以即使 plugin 有 bug，最壞情況只是「沒路由」，**絕不會打斷或破壞 gateway 的訊息流**。

**自動切模型**：透過 `gateway._session_model_overrides[session_key] = {...}` 設定該對話的模型覆蓋
（與 `/model` 指令同一機制），全程 best-effort + try/except，失敗也不影響。

### 3-2. recipes.yaml（配方庫，逐步擴充）

每個「配方」= 觸發詞 + 綁定模型 + 展開指令：
```yaml
recipes:
  - name: presentation
    triggers: ["簡報", "投影片", "週報", "報告給老闆", "slides", "ppt", ...]
    model: qwen3-coder:30b
    expand: |
      （完整的指令式 prompt：走 ppt-build-pipeline、讀檔全文不可當空模板、
        每單位一頁、SVG→safe_convert→qa_check、禁止 python-pptx、
        版面鐵則防大字遮蔽、效率鐵則防 context 暴脹…）
```

**擴充新任務 = 在 recipes.yaml 加一筆**（不需改程式）：
- 🟢 輕配方：執行能力已存在（翻譯、會議記錄、email 草稿…）→ 幾行搞定
- 🟡 重配方：需要全新執行能力（如簡報）→ 先建底層流程/工具，配方才薄薄一層

> 改完 recipes.yaml 需 **重啟 gateway** 才重載（plugin 啟動時讀取一次並快取）。

### 3-3. 執行引擎（先前已建，配方會呼叫）

- **設計風格 skills**：`~/.hermes/skills/ppt-design/`（7 種：futuristic-tech、exec-dark-dashboard 等）
- **ppt-build-pipeline skill**：標準製作流程（讀檔→大綱→SVG→轉檔→校稿）
- **ppt-master**：`~/.hermes/ppt-master/`，SVG→可編輯 PPTX 引擎（svg_to_pptx.py）
- **safe_convert.py**：防呆轉檔（自動修裸 `&`、驗 XML、不退陽春版）
- **qa_check.py**：確定性品質檢查 —— 對比、溢出、超界、**大字重疊**、豐富度
  - ⚠️ **transform-aware**：正確處理 `<g transform="translate()">` 群組的相對座標
    （累積位移算絕對座標後才檢查，否則全失準）

---

## 4. 完整運作流程

```
① 使用者在 Telegram：傳 Word 檔 + 打「幫我做成給老闆的簡報」
        │
        ▼
② gateway 收到 inbound message（run.py _handle_message）
        │  觸發 pre_gateway_dispatch hook
        ▼
③ task-router plugin：
   - 比對 recipes.yaml 觸發詞 → 命中「presentation」配方
   - gateway._session_model_overrides ← 切到 qwen3-coder:30b
   - 把訊息改寫成完整指令式 prompt（含使用者原話）
   - 回傳 {"action":"rewrite", "text": 完整指令}
        │
        ▼
④ agent 收到「已展開的完整任務」，用 qwen3-coder 執行：
   讀檔(markitdown) → 規劃 → 逐張產 SVG(遵守版面/效率鐵則)
   → safe_convert.py 轉 PPTX → qa_check.py 檢查到 0 問題
        │
        ▼
⑤ 把 PPTX 檔回傳給使用者
```

未命中任何配方（如一般聊天「今天天氣？」）→ plugin 回 None → 照常用預設模型聊天，完全不受影響。

---

## 5. 安全網與回滾

| 層級 | 保護 |
|------|------|
| 核心程式 | **完全沒動**（plugin 方式）→ 原本功能不可能壞 |
| plugin 本身 | callback 全程 try/except，出錯只是「不路由」，不破壞訊息流 |
| 一鍵停用 | `hermes plugins disable task-router` + `sudo systemctl restart hermes-gateway` |
| 完整還原 | 備份在 `~/hermes-backups/` 與 Mac `~/Desktop/Hermes備份/`，`tar xzf hermes-core.tar.gz` |

---

## 6. 日常使用與維運

### 使用（Telegram）
傳素材 + 一句含觸發詞的話（簡報/報告/週報/投影片…）→ 自動產出。
一般聊天不含觸發詞 → 照常（用 nemotron3 多模態）。

### 新增任務能力
1. 編輯 `~/.hermes/plugins/task-router/recipes.yaml`，加一筆 recipe（triggers / model / expand）
2. `sudo systemctl restart hermes-gateway` 重載
3. （重配方）若需要全新執行能力，先建對應 skill/工具/流程

### 常用指令
```bash
hermes plugins list                         # 看 plugin 狀態
hermes plugins enable/disable task-router   # 開關路由
sudo systemctl restart hermes-gateway       # 重啟(載入 plugin/recipes 變更)
tail -f ~/.hermes/logs/agent.log            # 看路由是否命中、模型是否切換
```

---

## 7. 已知限制（誠實說明）

1. **意圖偵測用關鍵字** → 講法太迂迴（完全不提觸發詞）會漏判，走一般聊天。可加 gemma4 兜底分類。
2. **執行品質受地端 30B 模型天花板限制** → 路由保證「用對流程」，但「做得多漂亮」仍約 80% Cowork 水準。
3. **配方需逐步累積** → 系統只對「有寫配方」的任務可靠；其餘走一般 agent。
4. **大任務速度** → 地端產多頁簡報需數分鐘（非秒回），配方已加效率鐵則減緩 context 暴脹。

---

## 附錄：關鍵檔案位置一覽

| 用途 | 路徑 |
|------|------|
| 路由 plugin | `~/.hermes/plugins/task-router/{plugin.yaml,__init__.py,recipes.yaml}` |
| 設計風格 skills | `~/.hermes/skills/ppt-design/` |
| 製作流程 skill | `~/.hermes/skills/ppt-design/ppt-build-pipeline/SKILL.md` |
| 轉檔/檢查工具 | `~/.hermes/ppt-master/{safe_convert.py,qa_check.py}` + `~/.hermes/ppt-master-venv` |
| Hermes 核心(未動) | `/usr/local/lib/hermes-agent/`（hook 定義在 gateway/run.py、plugins.py）|
| 設定/密鑰 | `~/.hermes/config.yaml`、`~/.hermes/.env` |
| 備份 | `~/hermes-backups/`、Mac `~/Desktop/Hermes備份/` |
