# PLAN.md — claude-setting-manager 升級計畫

> 狀態：**最終版，準備開發**
> 日期：2026-03-21
> 分支：feat/more-options
> 版本：v4（經 25-agent 分析 + 需求確認後定稿）

---

## 一、專案定位

在現有 `claude-setting-manager` 基礎上**漸進式擴展**。

- **暫不改名** — 等多 LLM 功能上線且驗證需求後再考慮改名為 `llm-md-manager`
- 支援多 Base 合併、互動式 CLI、多 LLM 輸出、Skill 完整資料夾下載

---

## 二、功能規劃

### Feature 1：多 Base 選擇與合併

**目標：** 同一個 category 下可有多個 base template，用戶多選後依序合併。

**命名規則：**
```
template/instruction/sdet/
  00_base.md              # 通用 base
  00_base_python.md       # Python 專用 base
  00_base_docker.md       # Docker 專用 base
  00_base_go.md           # Go 專用 base
  sample_repo_1.md        # profile
  sample_repo_2.md        # profile
```

**合併順序（profile 在前，base 在後）：**
```
最終輸出 = sample_repo_1 + 00_base_python + 00_base_docker
           (profile)       (base1)          (base2)
```
- **與現有行為一致**：profile 內容在前，base 內容在後（不是 breaking change）
- 多個 base 按**顯示順序（字母序）**合併，不是按用戶選擇的先後順序（@clack/prompts multiselect 的行為）
- 各片段之間以 `\n\n---\n\n` 分隔（與現有行為一致）
- 空白或不存在的 base 檔案：warn 並跳過，不中斷流程

**互動選單（Phase 2 實作時加入）：**
```
◆ 請選擇 base template（可多選，按字母序合併）
│ ■ 00_base
│ ■ 00_base_python
│ □ 00_base_docker
│ □ 00_base_go
```

**config.json 變更：**
```json
{
  "type": "instruction",
  "category": "sdet",
  "profile": "sample_repo_1",
  "bases": ["00_base", "00_base_python"]
}
```

**程式碼影響：**
- `template_reader.js`：新增 `scan_bases(type, category)` 掃描 `00_base*.md`（嚴格匹配 `/^00_base(_[a-z0-9_]+)?\.md$/`）；`read_template()` 改為接受 `bases` 陣列參數，回傳 `{ specific, bases: [...] }`
- `template_merger.js`：新增 `merge_fragments(fragments[])` 函式，過濾空白片段後 join；保留舊 `merge_template()` 做向下相容 wrapper
- `config_manager.js`：`create_config()` 和 `replace_config()` 接受 `bases` 參數並存入 config
- `init.js`：解析 `--base` flag（逗號分隔），傳入 `bases` 陣列；呼叫 `merge_fragments([specific, ...bases])`；**傳 `bases` 給 `create_config()` / `replace_config()`**
- `update.js`：從 config 讀取 `bases`（經 normalize 後保證存在），傳入 `read_template()`；呼叫 `merge_fragments([specific, ...bases])`；**更新 import 為 `merge_fragments`**
- `index.js`：init command 新增 `--base <bases>` option
- `constant.js`：更新 `COMMAND_GUIDE` 加入 `--base` 範例
- `list.js`：base 和 profile 分開顯示（用 regex 區分 `00_base*` 和其他 `.md`）
- 向下相容：若 config 無 `bases` 欄位，預設 `["00_base"]`
- `--skip-base` 行為不變（跳過所有 base）
- `--base` flag 僅支援逗號分隔（`--base 00_base,00_base_python`），不支援重複 flag（commander.js 預設行為會丟值）
- `--base` flag 解析時 trim 每個值：`option.base.split(',').map(s => s.trim())`

**邊際情況處理：**
| 情況 | 處理方式 |
|------|---------|
| 用戶選 0 個 base | 等同 `--skip-base`，只輸出 profile |
| 選擇的 base 檔案不存在 | warn（顯示檔名）並跳過，繼續處理其他 base |
| `--skip-base` + `--base` 同時提供 | `--skip-base` 優先，忽略 `--base` |
| base 檔案內容為空 | 跳過，不產生空的 `---` 分隔符 |
| skill type 的 config | 不加 `bases` 欄位（skill 無 base 概念）|

---

### Feature 2：互動式 CLI（雙模式：互動 + flag）

**目標：** 無 subcommand 時進入互動選單；有 subcommand 時直接執行（保持腳本相容）。

**判斷邏輯：**
```
有 subcommand（init/update/list） → 直接執行（現有行為，不變）
無 subcommand → 進入互動選單
非 TTY 環境且無 subcommand → 報錯退出
```

**實作方式：** 在 `src/index.js` 使用 `program.action()` 捕獲無 subcommand 的情況，呼叫新模組 `src/command/interactive.js`。

**互動流程（3-5 步）：**
```
$ npx claude-setting-manager@latest

◆ 請選擇操作
│ ● Instruction（產生 LLM 指令檔）
│ ○ Skill（下載 Skill 套件）
│ ○ List（查看所有可用 template）
│ ○ Update（更新已安裝的 template）
│ ○ Force Latest Version（清除本工具的 npx 快取）

◆ 請選擇 category
│ ● sdet
│ ○ team1
│ ○ team2

◆ 請選擇 profile              ← Skill 操作時跳過此步
│ ● sample_repo_1
│ ○ sample_repo_2

◆ 請選擇 base template（可多選）   ← 僅 Instruction + 有 2+ base 時出現
│ ■ 00_base
│ □ 00_base_python
│ □ 00_base_go

◆ 請選擇目標 LLM（可多選）        ← Phase 3 實作後加入
│ ■ Claude (.claude/CLAUDE.md)
│ □ Gemini (.gemini/GEMINI.md)
│ □ Codex (AGENTS.md)
```

**各操作的互動差異：**
| 操作 | category | profile | bases | 額外步驟 |
|------|----------|---------|-------|---------|
| Instruction | ✅ 選擇 | ✅ 選擇 | ✅ 多選（2+ base 時）| 檔案已存在 → overwrite 確認 |
| Skill | ✅ 選擇 | ❌ 跳過 | ❌ 跳過 | 檔案已存在 → overwrite 確認 |
| List | ❌ 跳過 | ❌ 跳過 | ❌ 跳過 | 直接呼叫 handle_list，顯示結果 |
| Update | ❌ 跳過 | ❌ 跳過 | ❌ 跳過 | 直接執行既有 update 邏輯；若無 config → 顯示 error |
| Force Latest Version | ❌ 跳過 | ❌ 跳過 | ❌ 跳過 | 確認提示 → 清除快取 |

**Overwrite 確認邏輯：**
- 時機：**所有 prompt 完成後、寫入檔案前**
- 檢查目標路徑是否已存在檔案
- 存在 → 用 `@clack/prompts` confirm 問是否覆蓋
- 不存在 → 直接寫入
- 用戶拒絕覆蓋 → 顯示 "Operation cancelled" 並退出（exit 0）
- `--force` flag 等效於自動確認覆蓋

**Force Latest Version 實作：**
- 用 `npm config get cache` 取得跨平台 cache 路徑
- 掃描 `<cache>/_npx/*/package.json` 找到本工具的快取目錄
- 刪除該目錄（`fs.rmSync(dir, { recursive: true, force: true })`）
- 刪除前顯示確認提示（因為影響全域狀態）
- 顯示結果：刪除了什麼 + 下一步提示「請執行 npx claude-setting-manager@latest」
- 失敗時 warn，不中斷

**關鍵決策：**
- 單一 base 時自動選取，不出現選單
- 所有互動 prompt 用 `isCancel()` 處理 Ctrl+C，不寫入任何檔案；exit code 0
- **不支援 "Back" 回上一步**（@clack/prompts 不支援），用戶需 Ctrl+C 重來
- 空 template 目錄（無 category）→ 顯示 error 訊息並退出
- flag 模式完整保留：`--type`, `--category`, `--profile`, `--base`（逗號分隔）, `--force`, `--skip-base`
- 想用不同 bases 重新 init → 選 Instruction 重新走流程，觸發 overwrite 確認

**技術選型：** `@clack/prompts` ~0.9（tilde pin，搭配 commander.js）

**新增檔案：** `src/command/interactive.js`（export `run_interactive()`）

**非 TTY 防護：**
```js
if (!process.stdout.isTTY && !process.argv.slice(2).length) {
  logger.error('Non-interactive mode: please provide a subcommand (init/update/list) with flags.');
  process.exit(1);
}
```

---

### Feature 3：多 LLM 輸出（Adapter Pattern）

**目標：** 同一份 source template 可輸出到不同 LLM 的設定檔。

**支援的 LLM：**

| LLM | 輸出路徑 | 備註 |
|-----|---------|------|
| Claude | `.claude/CLAUDE.md` | 現有行為 |
| Gemini | `.gemini/GEMINI.md` | 新增 |
| Codex | `AGENTS.md`（專案根目錄）| 新增 |

**架構：**
```
src/adapter/
  index.js          # adapter registry，getAdapter(name) => adapter
  claude.js         # { name, displayName, getOutputPath, transform, detectExisting }
  gemini.js
  codex.js
```

**Adapter 介面：**
```js
class BaseAdapter {
  get name() { }                              // 'claude'
  get displayName() { }                       // 'Claude (.claude/CLAUDE.md)'
  getOutputPath(type, category) { }           // 動態路徑（skill 需要 category）
  transform(content) { return content; }      // identity by default，未來可加格式轉換
  detectExisting(projectRoot) { }             // 檢查目標檔案是否已存在
}
```

**程式碼影響：**
- 新增 `src/adapter/` 目錄（4 個檔案）
- `init.js` / `update.js`：抽取共用的 `write_output(adapter, content, type, category)` 到新的 `src/core/writer.js`；迴圈遍歷 `targets` 陣列，每個 target 透過 adapter 輸出
- `config_manager.js`：config entry 新增 `targets` 欄位（陣列），預設 `["claude"]`
- `constant.js`：原有的 `OUTPUT_CLAUDE_MD_FILE_PATH` 等常數由 adapter 取代
- 互動選單加入 LLM multiselect 步驟（Phase 2 已有框架，此處加入選項）
- init command 新增 `--target <targets>` flag（逗號分隔，預設 `claude`）

**衝突處理：**
- 若目標路徑已有手動建立的檔案（如手寫的 `AGENTS.md`），應警告用戶並提供 backup 選項
- backup 策略：複製為 `AGENTS.md.backup` 後覆蓋

**config.json 變更：**
```json
{
  "type": "instruction",
  "category": "sdet",
  "profile": "sample_repo_1",
  "bases": ["00_base", "00_base_python"],
  "targets": ["claude", "gemini"]
}
```

**向下相容：** config entry 無 `targets` 欄位時，預設 `["claude"]`。

---

### Feature 4：Skill 完整資料夾下載

**目標：** Skill 下載時，複製 template 資料夾內的**所有內容**（不限定只有 SKILL.md）。

**Template 結構（skill 資料夾內放什麼就下載什麼）：**
```
template/skill/
  professional1/
    SKILL.md
    reference/              # 可選
      some_guide.md
      api_spec.yaml
    scripts/                # 可選
      setup.sh
    any_other_file.txt      # 可選，任何額外檔案
  professional2/
    SKILL.md
```

**下載後的輸出（完整鏡像複製）：**
```
.claude/skills/<category>/
  SKILL.md
  reference/
    some_guide.md
    api_spec.yaml
  scripts/
    setup.sh
  any_other_file.txt
```

**邏輯：**
- Skill type 在 `init.js` / `update.js` 中走**獨立的 early-return 路徑**：不經過 `read_template` 內容讀取和 `merge_template`，直接 `fs.cpSync(src, dest, { recursive: true })` 複製整個資料夾
- `template_reader.js`：skill type 的 `read_template()` 改為回傳 `{ source_dir: '<path>' }` 而非檔案內容
- 目標資料夾已存在時：覆蓋同名檔案，保留目標中多出的檔案（`fs.cpSync` 預設行為）
- config.json entry 不變（仍存 type + category，無需追蹤個別檔案）

---

## 三、不做的事（明確排除）

- ❌ 不做 LLM 特化的 source template（用 adapter transform 處理差異）
- ❌ 不做 Cursor / Copilot / Windsurf 支援（架構預留但本次不做）
- ❌ 不做 plugin marketplace 上架
- ❌ 暫不改名
- ❌ 不做 npx cache 自動清除（改為互動選單中的手動 Force Latest Version 選項）

---

## 四、實作順序

```
Phase 1 — 多 Base 選擇與合併（Feature 1）        ≈ 4-6h / 1 個週末
  ├── 1.1 template_reader 新增 scan_bases() 掃描 00_base*.md
  ├── 1.2 template_merger 新增 merge_fragments() 多片段合併
  ├── 1.3 config_manager 新增 bases 陣列欄位 + normalize 向下相容
  ├── 1.4 init command：解析 --base flag + 傳 bases 給 create_config/replace_config
  ├── 1.5 update command：讀取 config.bases + 更新 import/呼叫
  ├── 1.6 constant.js：更新 COMMAND_GUIDE
  └── 1.7 list command：base 和 profile 分開顯示

Phase 2 — 互動式 CLI（Feature 2）                 ≈ 8-12h / 2-3 個週末
  ├── 2.1 加入 @clack/prompts 依賴（~0.9 tilde pin）
  ├── 2.2 新增 src/command/interactive.js + program.action() 無 subcommand 捕獲
  ├── 2.3 實作互動流程（operation → category → profile → bases multiselect）
  ├── 2.4 加入 List 選項（呼叫 handle_list）
  ├── 2.5 Skill 操作：跳過 profile/bases 步驟
  ├── 2.6 Overwrite 確認邏輯（所有 prompt 完成後、寫入前）
  ├── 2.7 Force Latest Version 選項（掃描 _npx 快取 + 確認 + 刪除）
  ├── 2.8 Update 選項（直接呼叫 handle_update）
  └── 2.9 非 TTY 防護 + Ctrl+C isCancel() 處理 + 空目錄 error

Phase 3 — 多 LLM 輸出（Feature 3）               ≈ 4-6h / 1 個週末
  ├── 3.1 建立 src/adapter/ 目錄（index.js, claude.js, gemini.js, codex.js）
  ├── 3.2 抽取 src/core/writer.js（共用寫入邏輯）
  ├── 3.3 init/update command 改為迴圈遍歷 targets + adapter 輸出
  ├── 3.4 config_manager 新增 targets 欄位 + normalize 向下相容
  ├── 3.5 init command 新增 --target flag
  └── 3.6 互動選單加入 LLM multiselect 步驟

Phase 4 — Skill 完整資料夾下載（Feature 4）        ≈ 2-3h / 1 個晚上
  ├── 4.1 init.js / update.js：skill type 走 early-return + fs.cpSync
  └── 4.2 template_reader：skill type 回傳 { source_dir } 而非檔案內容

Phase 5 — 補測試                                   ≈ 8-12h / 2-3 個週末
  ├── 5.1 建立測試框架（vitest 或 node:test）
  ├── 5.2 unit test：config_manager（含 normalize）, template_reader, template_merger, adapter
  └── 5.3 integration test：init, update, list

Phase 6 — 收尾                                     ≈ 2-3h / 1 個晚上
  ├── 6.1 更新 README
  └── 6.2 版本號升級
```

**預估總工時：28-42h（7-10 個週末，約 2.5 個月 part-time）**

**依賴關係：**
```
Phase 1 (多 Base) ──→ Phase 2 (互動 CLI) ──→ Phase 3 (多 LLM) ──┬── Phase 5 (補測試) ── Phase 6 (收尾)
                                                                   │
                                              Phase 4 (Skill 下載) ─┘

Phase 1 → 2：互動 CLI 的 bases multiselect 依賴 scan_bases()
Phase 2 → 3：LLM multiselect 步驟加在互動流程中
Phase 4：獨立，可隨時插入（不依賴任何 Phase）
Phase 5：所有功能完成後統一補測試
```

---

## 五、技術依賴變更

| 依賴 | 用途 | 類型 |
|------|------|------|
| `@clack/prompts` (~0.9) | 互動式 CLI 選單 | 新增 production（Phase 2） |
| `vitest` 或 `node:test` | 測試框架 | 新增 devDependency（Phase 5） |
| `commander` | CLI 框架（保留） | 現有 |
| `chalk` | 終端顏色（保留） | 現有 |

---

## 六、Config 向下相容策略

| 情況 | 處理邏輯 |
|------|----------|
| config entry 無 `bases` 欄位 | 預設為 `["00_base"]` |
| config entry 有 `base`（字串） | 轉為 `bases: ["<value>"]`，刪除舊 `base` |
| config entry 已有 `bases`（陣列） | 原樣使用 |
| config entry 無 `targets` 欄位 | 預設為 `["claude"]` |
| config entry 已有 `targets`（陣列） | 原樣使用 |
| `bases` 中的檔案不存在 | warn 並跳過該 base |
| skill type 的 config entry | 不加 `bases`（skill 無 base 概念） |

遷移在 `config_manager.js` 的 `read_config_file()` 中透過 `normalize_config_entry()` 自動處理。
只做 in-memory 轉換，不回寫檔案。下次 `create_config`/`replace_config`/`update_timestamp` 時自然以新格式寫入。

---

## 七、已解決的問題

| 問題 | 決議 |
|------|------|
| 改名時機 | 延後，等多 LLM 功能上線後再改名 |
| Adapter transform | 初期全部 identity（內容不轉換），未來按需加格式轉換 |
| Config 遷移 | 讀取時 in-memory normalize，無需 migration script |
| npx cache | 不自動清除；互動選單提供「Force Latest Version」手動選項 |
| Skill 下載方式 | 整個資料夾 `fs.cpSync`，不逐檔處理，不經 merger |
| 互動 vs flag | 雙模式共存：有 subcommand → flag 模式，無 subcommand → 互動模式 |
| 合併順序 | profile 在前、base 在後（與現有行為一致，不是 breaking change）|
| Multiselect 排序 | 按顯示順序（字母序）合併，非選擇順序 |
| 多 config 寫同一檔案 | 實務上一個專案只有一個 instruction config，暫不處理 |
| `--base` flag 格式 | 僅支援逗號分隔，不支援重複 flag |
| 互動選單 Back 導航 | 不支援，@clack 無此功能，Ctrl+C 重來 |
| list 顯示 base vs profile | 用 regex 分開顯示 |
| Overwrite 確認 | 所有 prompt 完成後、寫入前檢查，已存在才確認 |

---

## 八、已知風險與接受決策

| 風險 | 等級 | 決策 |
|------|------|------|
| Phase 1-4 無測試就開發 | M | 接受，Phase 5 統一補；config normalize 邏輯建議寫完後立即手動測試 |
| @clack/prompts 0.x API 穩定性 | L | 接受，用 tilde pin (~0.9) 限制版本範圍 |
| Force Latest Version 跨平台 _npx 路徑 | M | 用 `npm config get cache` 動態取得，不硬編碼路徑 |
| base 檔案命名 glob 太寬鬆 | L | 用嚴格 pattern：`/^00_base(_[a-z0-9_]+)?\.md$/` |
| Adapter transform 維護成本 | M | 初期 identity，各 LLM 格式變更時再加 transform |
| AGENTS.md 衝突（手寫 vs 自動產生） | M | detectExisting + backup 策略 |
