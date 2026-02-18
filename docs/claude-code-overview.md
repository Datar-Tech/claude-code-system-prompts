# Claude Code 運作架構總覽

> 基於 Claude Code v2.1.45 系統提示的深度分析。

## 目錄

- [概覽](#概覽)
- [五大提示層級](#五大提示層級)
  - [核心身份](#1-核心身份main-system-prompt)
  - [行為準則](#2-行為準則system-prompts--30-個)
  - [工具描述](#3-工具描述tool-descriptions--23-個)
  - [子代理系統](#4-子代理系統agent-prompts--29-個)
  - [系統提醒](#5-系統提醒system-reminders--42-個)
- [關鍵設計特點](#關鍵設計特點)
- [調整方法](#調整-claude-code-的方法)

---

## 概覽

Claude Code 是 Anthropic 的 CLI 代理式編程工具，以編譯後的 npm 套件（`@anthropic-ai/claude-code`）發佈。它並非靠「一條系統提示詞」運作，而是由 **110+ 個獨立的提示片段** 根據不同的環境、設定與工具動態拼裝。

來自 README.md：

> *「Claude Code doesn't just have one single string for its system prompt. Instead, there are: Large portions conditionally added depending on the environment and various configs... The result — 110+ strings that are constantly changing and moving within a very large minified JS file.」*

這些提示片段包含模板變數（如 `${BASH_TOOL_NAME}`），在運行時插值。

---

## 五大提示層級

### 1. 核心身份（Main System Prompt）

來源：`system-prompt-main-system-prompt.md`

最基礎的一層，定義 Claude Code 是一個「互動式 CLI 工具，協助軟體工程任務」。包含：
- 安全政策（`${SECURITY_POLICY}`）
- 輸出風格設定（`${OUTPUT_STYLE_CONFIG}`）
- URL 生成限制
- 使用者回饋管道

### 2. 行為準則（System Prompts × 30 個）

控制 Claude Code 的行為模式，主要包括：

**Doing tasks**（`system-prompt-doing-tasks.md`）：
- 先讀再改，理解現有程式碼後再建議修改
- 避免過度工程——不加多餘功能、抽象、註釋
- 不為假設的未來需求設計
- 避免安全漏洞（OWASP top 10）
- 不加向下相容 hack，未使用的程式碼直接刪除

**Executing actions with care**（`system-prompt-executing-actions-with-care.md`）：
- 區分「可逆/本地」與「不可逆/共享」操作
- 破壞性操作（刪除檔案、force push、drop table）必須先確認
- 影響他人的操作（push 程式碼、發送訊息、建立 PR）必須先確認
- 遇到障礙時不用破壞性操作繞過，而是調查根因
- 使用者一次批准不代表所有情境都批准，授權範圍嚴格限定

**Tool usage policy**（`system-prompt-tool-usage-policy.md`）：
- 優先用專用工具（Read/Edit/Write）而非 bash 指令
- 獨立工具呼叫可並行，有依賴的需序列
- 不在 bash 中用 echo 和使用者溝通

**Tool permission mode**（`system-prompt-tool-permission-mode.md`）：
- 使用者可選擇權限等級
- 被拒絕的工具呼叫不重試，調整策略

**Context compaction**（`system-prompt-context-compaction-summary.md`）：
- 上下文接近 context limit 時自動壓縮
- 產生結構化摘要（任務概述、當前狀態、重要發現、下一步、保留上下文）

### 3. 工具描述（Tool Descriptions × 23 個）

每個工具都有獨立的系統提示指導使用方式：

| 工具 | 檔案 | 用途 |
|------|------|------|
| **Bash** | `tool-description-bash.md` | shell 命令執行，含超時、沙箱、目錄驗證規則 |
| **Bash (Git)** | `tool-description-bash-git-commit-and-pr-creation-instructions.md` | Git commit 和 PR 建立的詳細步驟與安全協議 |
| **Bash (Sandbox)** | `tool-description-bash-sandbox-note.md` | 沙箱模式、`dangerouslyDisableSandbox` 規則 |
| **Read** | `tool-description-readfile.md` | 檔案讀取（支援圖片、PDF、Jupyter） |
| **Edit** | `tool-description-edit.md` | 精確字串替換 |
| **Write** | `tool-description-write.md` | 檔案建立/覆寫 |
| **Glob** | `tool-description-glob.md` | 檔案模式匹配搜尋 |
| **Grep** | `tool-description-grep.md` | 內容正則搜尋 |
| **Task** | `tool-description-task.md` | 啟動子代理（含背景執行、恢復、並行調度） |
| **TodoWrite** | `tool-description-todowrite.md` | 任務追蹤管理 |
| **EnterPlanMode** | `tool-description-enterplanmode.md` | 進入計畫模式 |
| **ExitPlanMode** | `tool-description-exitplanmode.md` | 退出計畫模式，請求使用者審批 |
| **AskUserQuestion** | `tool-description-askuserquestion.md` | 向使用者提問 |
| **WebFetch** | `tool-description-webfetch.md` | 網頁內容擷取 |
| **WebSearch** | `tool-description-websearch.md` | 網頁搜尋 |
| **Skill** | `tool-description-skill.md` | 呼叫技能（slash commands） |
| **SendMessage** | `tool-description-sendmessagetool.md` | Agent Teams 訊息 |
| **TeamCreate** | `tool-description-teammatetool.md` | Agent Teams 建立 |
| **TeamDelete** | `tool-description-teamdelete.md` | Agent Teams 清理 |
| **TaskCreate** | `tool-description-taskcreate.md` | 結構化任務建立 |
| **NotebookEdit** | `tool-description-notebookedit.md` | Jupyter notebook 編輯 |

### 4. 子代理系統（Agent Prompts × 29 個）

Claude Code 可以啟動獨立的子代理來處理特定任務。詳見 [claude-code-subagent-architecture.md](./claude-code-subagent-architecture.md)。

主要分類：
- **核心子代理**：Explore（唯讀探索）、Plan（架構規劃）、Bash（命令執行）、general-purpose（通用）
- **隱形功能子代理**：對話摘要、情緒分析、命令前綴偵測、Session 記憶更新等
- **Slash Command 子代理**：/review-pr、/security-review、/pr-comments
- **建立輔助子代理**：CLAUDE.md 建立、Agent 建立架構師、Status line 設定

### 5. 系統提醒（System Reminders × 42 個）

在對話過程中根據狀況動態注入的短提示：

| 類別 | 範例 |
|------|------|
| 計畫模式 | 5-phase plan mode、iterative plan mode、subagent plan mode、plan re-entry |
| 檔案狀態 | 檔案被外部修改、檔案為空、檔案被截斷 |
| Hook 系統 | hook 成功、hook 阻止、hook 附加上下文 |
| 任務管理 | TodoWrite 提醒、任務狀態更新 |
| 團隊協調 | team coordination、team shutdown、delegate mode |
| 資源監控 | token 用量、USD 預算、輸出 token 超限 |
| 安全 | 惡意軟體分析提醒、診斷偵測 |

---

## 關鍵設計特點

| 特點 | 說明 |
|------|------|
| **條件式組裝** | 提示透過模板變數和條件邏輯在運行時動態拼裝 |
| **階層式權限** | 使用者選擇權限模式，工具被拒絕後自動調整策略 |
| **上下文壓縮** | 接近 context limit 時自動產生結構化摘要，延續工作不中斷 |
| **子代理隔離** | Task tool 啟動獨立代理，保護主 context window |
| **安全優先** | 破壞性操作必確認、不跳過 hooks、不 force push、不猜 URL |
| **最小變更原則** | 不過度工程、不加多餘註釋/docstring、不為假設需求設計 |

---

## 調整 Claude Code 的方法

從最輕量到最深度排列：

### 方法 1：CLAUDE.md（官方推薦，最簡單）

在專案根目錄建立 `CLAUDE.md`，寫入專案特定指令。Claude Code 自動讀取並遵循。

```markdown
# CLAUDE.md
- 使用 TypeScript strict mode
- commit message 用中文
- 測試框架用 vitest
```

### 方法 2：Hooks 設定

在 `~/.claude/` 設定 hooks，在工具呼叫前後執行 shell 命令，用於自動化流程或自訂檢查。支援 command、prompt、agent 三種 hook 類型。

### 方法 3：Skills（技能系統）

透過 SKILL.md 定義可重複的工作流，用 slash commands（如 `/commit`、`/review-pr`）觸發。可自訂新技能。

### 方法 4：tweakcc（深度客製化）

Piebald AI 提供的工具，直接修改 Claude Code 的系統提示。詳見 [claude-code-tweakcc-customization.md](./claude-code-tweakcc-customization.md)。

### 方法 5：追蹤版本變化

利用本倉庫的 CHANGELOG.md 追蹤每個版本的提示詞變動，了解 Anthropic 的設計意圖與演進方向。
