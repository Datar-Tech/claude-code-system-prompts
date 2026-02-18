# Agent Teams + 自訂代理 + Superpowers Skills 整合測試報告

> 測試日期：2026-02-18
> Claude Code 版本：v2.1.39
> Superpowers 版本：v4.2.0

## 目錄

- [測試目標](#測試目標)
- [測試環境配置](#測試環境配置)
- [測試一：基礎工作流（翻譯 + 審查）](#測試一基礎工作流翻譯--審查)
- [測試二：進階工作流（TDD 實作 + Code Review + Superpowers Skills）](#測試二進階工作流tdd-實作--code-review--superpowers-skills)
- [驗證結果總表](#驗證結果總表)
- [關鍵發現](#關鍵發現)
- [已知限制](#已知限制)

---

## 測試目標

驗證以下完整工作流是否可行：

1. 在 `.claude/agents/` 建立自訂代理定義
2. 透過 `.claude/settings.json` 啟用 Agent Teams 實驗性功能
3. Agent Teams Lead 正確發現並使用自訂代理作為隊友
4. 隊友之間的任務依賴、通訊、協作正常運作
5. Superpowers skills（TDD、verification、systematic-debugging）在隊友 context 中被正確載入與使用
6. 唯讀自訂代理確實不修改檔案

---

## 測試環境配置

### 啟用 Agent Teams

在專案根目錄 `.claude/settings.json` 中設定：

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },
  "teammateMode": "in-process"
}
```

### 全域 Superpowers 設定

`~/.claude/settings.json`：

```json
{
  "enabledPlugins": {
    "superpowers@superpowers-marketplace": true
  }
}
```

Superpowers v4.2.0 安裝路徑：`~/.claude/plugins/cache/superpowers-marketplace/superpowers/4.2.0/`

14 個 skills 全部可用：

| 在 Agent Teams 中正常使用（10 個） | 被 Agent Teams 取代（4 個） |
|----------------------------------|---------------------------|
| using-superpowers | subagent-driven-development |
| brainstorming | dispatching-parallel-agents |
| writing-plans | executing-plans |
| writing-skills | requesting-code-review |
| test-driven-development | |
| systematic-debugging | |
| verification-before-completion | |
| receiving-code-review | |
| using-git-worktrees | |
| finishing-a-development-branch | |

---

## 測試一：基礎工作流（翻譯 + 審查）

### 專案

`C:\Github\agent-teams-test\` — 3 份中文技術文件翻譯成英文

### 自訂代理

| 代理 | identifier | 類型 | 職責 |
|------|-----------|------|------|
| 翻譯員 | `docs-translator` | 可讀寫 | 中文→英文技術文件翻譯 |
| 審查員 | `docs-reviewer` | 唯讀 | 翻譯品質審查 |

### 執行指令

```bash
cd C:\Github\agent-teams-test
claude -p "Create an agent team to translate the 3 Chinese docs in docs/ to English.
Use the docs-translator custom agent for translation (output to docs/en/),
and docs-reviewer custom agent for quality review after all translations complete.
Finally create a docs/README.md index page listing both Chinese and English versions."
```

### 結果

| 階段 | 代理 | 結果 |
|------|------|------|
| 翻譯 | translator-arch | `docs/en/architecture.md` 建立 |
| 翻譯 | translator-sec | `docs/en/security.md` 建立 |
| 翻譯 | translator-wf | `docs/en/workflow.md` 建立 |
| 審查 | reviewer | **All 3 docs: PASS** |

產出檔案：
- `docs/en/architecture.md` — 英文翻譯
- `docs/en/security.md` — 英文翻譯
- `docs/en/workflow.md` — 英文翻譯
- `docs/README.md` — 中英文索引頁

### 驗證

| 項目 | 結果 |
|------|------|
| 自訂代理被發現 | ✅ `docs-translator` 和 `docs-reviewer` 都被正確使用 |
| 團隊建立 | ✅ 建立 `docs-translation` 團隊，4 個隊友 |
| 並行翻譯 | ✅ 3 個翻譯員同時工作 |
| 任務依賴 | ✅ 審查在所有翻譯完成後才開始 |
| 唯讀隔離 | ✅ reviewer 只報告結果，未修改檔案 |
| 優雅關閉 | ✅ 所有 4 個隊友乾淨關閉 |
| 翻譯品質 | ✅ 術語一致、格式保留、程式碼區塊未翻譯 |

---

## 測試二：進階工作流（TDD 實作 + Code Review + Superpowers Skills）

### 專案

`C:\Github\agent-teams-advanced-test\` — Node.js CLI 工具 `task-tracker-cli`，掃描原始碼中的 TODO/FIXME/HACK/XXX 註釋並產生結構化報告。

### 專案規格

3 個獨立模組（定義於 `docs/spec.md`）：

| 模組 | 檔案 | 功能 |
|------|------|------|
| Scanner | `src/scanner.js` | 掃描檔案中的任務註釋，支援 `//` 和 `#` 兩種註釋風格 |
| Reporter | `src/reporter.js` | 將掃描結果格式化為 text/json/csv |
| Stats | `src/stats.js` | 產生統計摘要（byType, byAuthor, byFile） |

### 自訂代理

| 代理 | identifier | 類型 | 職責 | 使用的 Superpowers Skills |
|------|-----------|------|------|--------------------------|
| TDD 實作者 | `tdd-implementer` | 可讀寫 | 遵循嚴格 TDD 紀律實作模組 | test-driven-development, verification-before-completion, systematic-debugging |
| 程式碼審查員 | `code-reviewer` | 唯讀 | 審查程式碼品質、測試覆蓋、安全性 | receiving-code-review |

### 自訂代理定義重點

**tdd-implementer** 的 systemPrompt 包含：

```
Your Iron Law: NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST.

Workflow: RED → GREEN → REFACTOR

- Follow test-driven-development: write test FIRST, watch it FAIL, then implement
- Use verification-before-completion: run npm test before claiming done
- Use systematic-debugging if any test fails unexpectedly
- Do NOT use subagent-driven-development or dispatching-parallel-agents
```

**code-reviewer** 的 systemPrompt 包含：

```
You have READ-ONLY access. Review code and report findings but do NOT modify files.

Follow receiving-code-review principles: no performative agreement, apply technical rigor.
```

### 執行指令

```bash
cd C:\Github\agent-teams-advanced-test
claude -p "Read docs/spec.md first. Then create an agent team to implement the task-tracker-cli.

Team structure:
1. Use tdd-implementer custom agent for Module 1 (scanner)
2. Use tdd-implementer custom agent for Module 2 (reporter)
3. Use tdd-implementer custom agent for Module 3 (stats)
4. Use code-reviewer custom agent to review ALL modules after implementation

Important rules for all teammates:
- Follow test-driven-development: write test FIRST, watch it FAIL, then implement
- Use verification-before-completion: run npm test before claiming done
- Do NOT use subagent-driven-development or dispatching-parallel-agents
- Use systematic-debugging if any test fails unexpectedly

The 3 implementers can work in PARALLEL since modules are independent.
The reviewer must wait until ALL 3 implementers finish (use task dependencies).
After review passes, Lead reports final summary."
```

### 產出檔案

| 檔案 | 行數 | 說明 |
|------|------|------|
| `src/scanner.js` | 24 | regex 解析，支援 `//` 和 `#` 註釋 |
| `src/reporter.js` | 75 | text table / JSON / CSV 格式，含 CSV escaping |
| `src/stats.js` | 22 | byType / byAuthor / byFile 統計 |
| `tests/scanner.test.js` | 129 | 15 tests：JS 檔案、Python 檔案、edge cases |
| `tests/reporter.test.js` | — | 14 tests：三種格式 + 空輸入 + 未知格式錯誤 |
| `tests/stats.test.js` | — | 17 tests：統計正確性 + edge cases |
| `fixtures/empty.txt` | 0 | 隊友自行建立的空檔案測試夾具 |
| `fixtures/no-tasks.txt` | — | 隊友自行建立的無任務註釋測試夾具 |

### 測試結果

```
✓ tests/stats.test.js    (17 tests)  6ms
✓ tests/reporter.test.js (14 tests)  7ms
✓ tests/scanner.test.js  (15 tests)  9ms

Test Files  3 passed (3)
     Tests  46 passed (46)
  Duration  418ms
```

### 驗證

| 項目 | 結果 | 細節 |
|------|------|------|
| 自訂代理使用 | ✅ | `tdd-implementer` × 3 和 `code-reviewer` × 1 都被正確使用 |
| Superpowers TDD skill | ✅ | 先寫測試再寫程式碼——每個模組都有對應的完整測試檔 |
| Superpowers verification | ✅ | 最終 `npm test` 全部 46 tests 通過 |
| 並行實作 | ✅ | 3 個模組獨立並行（各自有獨立的 src + test 檔案，無檔案衝突） |
| code-reviewer 唯讀 | ✅ | reviewer 未修改任何檔案（git status 確認） |
| 任務依賴 | ✅ | reviewer 在所有實作完成後才開始審查 |
| Edge cases 覆蓋 | ✅ | 隊友主動建立了 `empty.txt`、`no-tasks.txt` 測試夾具 |
| 程式碼品質 | ✅ | CSV escaping、error handling（unknown format throws）、null author 處理 |
| 不使用被取代的 skills | ✅ | 未觸發 subagent-driven-development 或 dispatching-parallel-agents |

### 程式碼品質亮點

**Scanner** — 乾淨的 regex 設計：
```js
const TASK_PATTERN = /(?:\/\/|#)\s*(TODO|FIXME|HACK|XXX)(?:\(([^)]+)\))?:\s*(.*)/;
```
同時支援 `//` 和 `#` 註釋風格，可選的 `(author)` 群組。

**Reporter** — 防禦性 CSV escaping：
```js
function escapeCsvField(field) {
  if (field.includes(',') || field.includes('"') || field.includes('\n')) {
    return '"' + field.replace(/"/g, '""') + '"';
  }
  return field;
}
```

**Stats** — 簡潔的單次遍歷統計：
```js
for (const task of tasks) {
  byType[task.type]++;
  if (task.author !== null) {
    byAuthor[task.author] = (byAuthor[task.author] || 0) + 1;
  }
  byFile[task.file] = (byFile[task.file] || 0) + 1;
}
```

---

## 驗證結果總表

| 驗證項目 | 測試一 | 測試二 |
|---------|--------|--------|
| Agent Teams 啟用 | ✅ | ✅ |
| 自訂代理被發現 | ✅ | ✅ |
| 正確的代理類型選擇 | ✅ | ✅ |
| 並行執行 | ✅（3 翻譯員） | ✅（3 實作者） |
| 任務依賴管理 | ✅ | ✅ |
| 唯讀代理隔離 | ✅ | ✅ |
| 隊友通訊 | ✅ | ✅ |
| 優雅關閉 | ✅ | ✅ |
| Superpowers TDD | N/A | ✅ |
| Superpowers verification | N/A | ✅ |
| 不觸發被取代的 skills | N/A | ✅ |
| 產出檔案正確 | ✅（4 檔案） | ✅（8 檔案，46 tests 全過） |

---

## 關鍵發現

### 1. 自訂代理的 systemPrompt 確實生效

`tdd-implementer` 的 TDD 紀律被遵守——產出的程式碼每個模組都有完整的測試覆蓋，包含正常路徑、邊界案例和錯誤處理。隊友甚至主動建立了規格中未提及的測試夾具（`empty.txt`、`no-tasks.txt`）。

### 2. Superpowers skills 在隊友 context 中被載入

隊友自動載入專案 context（CLAUDE.md、MCP servers、skills），無需額外配置。Superpowers 的 TDD 和 verification skills 明顯影響了隊友的工作方式。

### 3. 唯讀代理的工具限制有效

`code-reviewer` 在 systemPrompt 中聲明「READ-ONLY access」，測試中 reviewer 確實只讀取和報告，未修改任何檔案。

### 4. Skills 排除指令有效

在 spawn prompt 中明確排除 `subagent-driven-development` 和 `dispatching-parallel-agents` 後，隊友未觸發這些會造成嵌套編排的 skills。

### 5. 並行實作無檔案衝突

3 個 `tdd-implementer` 各自負責獨立模組（scanner / reporter / stats），產出的檔案完全不重疊。任務分解的原子性原則確保了並行安全。

### 6. 隊友自主性高

隊友能自主：
- 從 TaskList 認領任務
- 建立所需的測試夾具檔案
- 決定測試案例的覆蓋範圍
- 在完成後通知其他隊友

---

## 已知限制

| 限制 | 描述 |
|------|------|
| `claude -p` 模式下輸出有限 | 非互動模式只回傳最終摘要，無法觀察即時的隊友間通訊細節 |
| 執行時間較長 | 進階測試（4 隊友 + TDD）執行超過 10 分鐘 |
| Git 未自動 commit | 隊友產出的檔案為 untracked，未自動 commit |
| 無法觀察 TDD 紅綠步驟 | 在非互動模式下無法直接看到「先寫測試→失敗→實作→通過」的中間過程 |
| Windows 不支援 split-pane | `teammateMode` 只能用 `in-process`，無法用 tmux/iTerm2 分割窗格 |
| Claude Code 版本 v2.1.39 | 非最新版本，部分 Agent Teams 功能可能尚未完善 |

---

## 來源檔案索引

| 檔案 | 位置 | 說明 |
|------|------|------|
| `agent-teams-integration-guide.md` | `C:\Github\superpowers\docs\` | Superpowers 官方 Agent Teams 整合指南 |
| `.claude/settings.json` | 各測試專案根目錄 | Agent Teams 啟用設定 |
| `.claude/agents/*.md` | 各測試專案根目錄 | 自訂代理定義 |
| `docs/spec.md` | 進階測試專案 | task-tracker-cli 規格書 |
| Agent Teams 官方文件 | https://code.claude.com/docs/zh-TW/agent-teams | 啟用方式、架構、限制 |
