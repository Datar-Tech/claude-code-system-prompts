# tweakcc 自訂 Claude Code 指南

> 基於 Claude Code v2.1.45 系統提示與 tweakcc 工具的深度分析。

## 目錄

- [tweakcc 概覽](#tweakcc-概覽)
- [技術原理](#技術原理)
- [基本工作流程](#基本工作流程)
- [七大自訂情境](#七大自訂情境)
  - [1. 自訂 Explore 子代理](#1-自訂-explore-子代理)
  - [2. 自訂 Plan 子代理](#2-自訂-plan-子代理)
  - [3. 修改命令前綴偵測](#3-修改命令前綴偵測)
  - [4. 調整 Task 工具調度行為](#4-調整-task-工具調度行為)
  - [5. 擴展安全審查](#5-擴展安全審查)
  - [6. 修改 Git 安全協議](#6-修改-git-安全協議)
  - [7. 調整沙箱規則](#7-調整沙箱規則)
- [adhoc-patch 進階技術](#adhoc-patch-進階技術)
- [團隊級配置分發](#團隊級配置分發)
- [版本更新與衝突管理](#版本更新與衝突管理)
- [三種自訂路線比較](#三種自訂路線比較)

---

## tweakcc 概覽

[tweakcc](https://github.com/Piebald-AI/tweakcc) 是 Piebald AI 開發的 CLI 工具，用於直接修改 Claude Code 安裝中的系統提示。它允許使用者以 markdown 檔案的形式自訂個別提示片段，然後將修改注入 Claude Code 的編譯後原始碼。

**核心定位**：tweakcc 是目前唯一能直接修改 Claude Code 系統提示的工具——CLAUDE.md 是「追加指令」，tweakcc 是「替換底層提示」。

---

## 技術原理

### 修改目標

Claude Code 的所有系統提示以模板字串形式嵌入在編譯後的 JavaScript 檔案（`cli.js`）中。每個提示有：

- **pieces**：模板字串的靜態文字片段陣列
- **identifiers**：模板變數的 ID 陣列
- **identifierMap**：ID 到變數名的對映

tweakcc 的核心操作是：
1. 讀取編譯後的 `cli.js`
2. 定位目標提示的模板字串
3. 用修改後的內容替換
4. 寫回 `cli.js`

### 支援的安裝類型

| 安裝類型 | 修改方式 |
|---------|---------|
| **npm 安裝** | 直接修改 `node_modules/@anthropic-ai/claude-code/cli.js` |
| **原生二進位** | 透過 `node-lief` 解包二進位檔案，修改後重新封裝 |

---

## 基本工作流程

### 1. 匯出現有提示

```bash
tweakcc export
```

將所有系統提示匯出為獨立的 markdown 檔案到工作目錄。每個檔案包含 YAML frontmatter（名稱、描述、版本、模板變數）和提示內容。

### 2. 修改提示

直接編輯匯出的 markdown 檔案。模板變數以 `${VARIABLE_NAME}` 的形式保留在文中。

### 3. 注入修改

```bash
tweakcc patch
```

將修改過的檔案注入回 Claude Code 安裝。

### 4. 驗證

```bash
tweakcc diff
```

查看當前修改與原始提示的差異。

### 5. 還原

```bash
tweakcc restore
```

恢復為原始提示。

---

## 七大自訂情境

### 1. 自訂 Explore 子代理

**目標檔案**：`agent-prompt-explore.md`（516 tks）

**可自訂內容**：
- 修改搜尋策略——例如強制優先搜尋特定目錄
- 調整唯讀限制——例如允許特定的 bash 命令
- 變更輸出格式——例如要求結構化 JSON 輸出

**範例：加入專案特定搜尋路徑**：

在原始提示的搜尋策略區段後加入：

```markdown
## Project-Specific Search Paths
Always search these directories first:
- src/core/ - Core business logic
- src/api/ - API endpoints
- tests/ - Test files
```

**注意**：Explore 子代理的唯讀限制（`=== CRITICAL: READ-ONLY MODE ===`）是安全關鍵。移除此限制可能導致安全問題。

### 2. 自訂 Plan 子代理

**目標檔案**：`agent-prompt-plan-mode-enhanced.md`（633 tks）

**可自訂內容**：
- 修改規劃流程——例如加入架構決策記錄（ADR）格式
- 調整「Critical Files」清單要求——例如從 3-5 個增加到更多
- 加入專案特定的架構約束

**範例：要求 ADR 格式輸出**：

```markdown
## Output Format
Your plan must include an Architecture Decision Record (ADR):
- **Status**: Proposed
- **Context**: Why this change is needed
- **Decision**: What we decided to do
- **Consequences**: What are the trade-offs
```

### 3. 修改命令前綴偵測

**目標檔案**：`agent-prompt-bash-command-prefix-detection.md`（823 tks）

**可自訂內容**：
- 新增自訂命令的前綴提取規則
- 調整注入偵測的靈敏度
- 加入組織特定的命令模式

**範例：加入內部工具的前綴規則**：

```markdown
## Additional prefix extraction examples
- deploy staging => deploy staging
- deploy production => none
- internal-tool scan --deep => internal-tool scan
- k8s apply -f manifest.yaml => k8s apply
```

**警告**：降低注入偵測靈敏度是高風險操作。建議只增加規則，不移除現有偵測模式。

### 4. 調整 Task 工具調度行為

**目標檔案**：`tool-description-task.md`（1214 tks）

**可自訂內容**：
- 修改子代理選擇的優先順序
- 調整並行啟動的策略
- 加入自訂子代理類型的描述

**範例：優先使用自訂代理**：

在代理類型列表中加入：

```markdown
- security-checker: Security specialist for reviewing code changes.
  Use this agent after completing any code modification to check
  for security vulnerabilities. (Tools: Read, Grep, Glob, Bash)
```

### 5. 擴展安全審查

**目標檔案**：`agent-prompt-security-review-slash-command.md`（2610 tks）

這是最大的子代理提示，包含三階段安全審查流程和 17 條誤報排除規則。

**可自訂內容**：
- 加入組織特定的安全檢查項目
- 修改 confidence 閾值（預設為 ≥ 8）
- 加入自訂的誤報排除規則
- 修改漏洞分類標準

**範例：加入 OWASP Top 10 檢查清單**：

```markdown
## Additional Security Checks
For each file, also verify against OWASP Top 10 (2021):
1. A01: Broken Access Control
2. A02: Cryptographic Failures
3. A03: Injection
...
```

### 6. 修改 Git 安全協議

**目標檔案**：`tool-description-bash-git-commit-and-pr-creation-instructions.md`（1608 tks）

**可自訂內容**：
- 修改 commit message 格式——例如強制使用 Conventional Commits
- 調整 PR 建立流程——例如加入自動標籤
- 放寬或收緊特定規則

**範例：強制 Conventional Commits**：

```markdown
## Commit Message Format
ALWAYS use Conventional Commits format:
- feat: A new feature
- fix: A bug fix
- docs: Documentation only changes
- style: Code style changes
- refactor: Code refactoring
- test: Adding tests
- chore: Build process or tool changes

Example: feat(auth): add OAuth2 login support
```

**注意**：Git Safety Protocol 中的 NEVER 規則是安全關鍵。移除這些規則可能導致資料遺失。

### 7. 調整沙箱規則

**目標檔案**：`tool-description-bash-sandbox-note.md`（438 tks）

**可自訂內容**：
- 修改沙箱失敗判斷條件
- 調整自動重試策略
- 修改敏感路徑清單

**範例：加入自動允許的路徑**：

```markdown
## Project-Specific Allowed Paths
The following paths are safe to access outside sandbox:
- /opt/company-tools/ - Internal development tools
- /var/data/test-fixtures/ - Test data directory
```

**警告**：這是高風險修改。沙箱的反模式學習規則是重要安全機制，不建議移除。

---

## adhoc-patch 進階技術

tweakcc 除了修改完整的提示檔案外，還支援 adhoc-patch 模式，可以對 `cli.js` 進行更底層的修改。

### 三種 adhoc-patch 模式

#### 1. String 模式

直接字串替換：

```json
{
  "type": "string",
  "find": "NEVER update the git config",
  "replace": "Avoid updating the git config unless requested"
}
```

- **優點**：簡單直觀
- **風險**：如果字串在多處出現，可能產生非預期的替換

#### 2. Regex 模式

正規表達式替換：

```json
{
  "type": "regex",
  "pattern": "NEVER run destructive git commands \\([^)]+\\)",
  "replacement": "Avoid running destructive git commands ($1) unless explicitly requested with confirmation"
}
```

- **優點**：精確匹配
- **風險**：regex 錯誤可能導致修改失敗或損壞

#### 3. Script 模式

在沙箱化環境中執行修改腳本：

```json
{
  "type": "script",
  "script": "module.exports = (source) => source.replace(/pattern/g, 'replacement')"
}
```

- **優點**：最大靈活性，可以進行複雜的條件修改
- **風險**：腳本在沙箱中執行，但仍需謹慎

---

## 團隊級配置分發

### 共享配置方式

1. **版本控制**：將修改後的提示檔案和 tweakcc 配置提交到團隊共享的 repository
2. **自動化腳本**：用 CI/CD 或 setup script 在團隊成員的機器上自動執行 `tweakcc patch`
3. **CLAUDE.md 配合**：tweakcc 修改底層提示，CLAUDE.md 追加專案指令，兩者互補

### 配置結構範例

```
team-claude-config/
├── prompts/                        # 修改後的提示檔案
│   ├── agent-prompt-explore.md
│   ├── agent-prompt-plan-mode-enhanced.md
│   └── tool-description-bash-git-commit-and-pr-creation-instructions.md
├── adhoc-patches/                  # adhoc 修補檔
│   └── custom-rules.json
├── setup.sh                        # 自動化安裝腳本
└── README.md                       # 團隊配置說明
```

---

## 版本更新與衝突管理

### Claude Code 更新時的流程

```
Claude Code 新版本發佈
        │
        ▼
tweakcc diff  ← 比較修改與新版本
        │
        ├── 無衝突 → tweakcc patch → 重新注入修改
        │
        └── 有衝突 → 手動解決
                      │
                      ├── Anthropic 修改了相同的提示
                      ├── 模板變數改名
                      └── 提示結構重組
```

### 衝突解決策略

| 衝突類型 | 建議策略 |
|---------|---------|
| Anthropic 改進了同一區域 | 評估 Anthropic 的改動是否更優，考慮合併 |
| 模板變數改名 | 更新修改檔案中的變數名 |
| 提示結構大幅重組 | 重新匯出，基於新結構重做修改 |
| 新增的安全規則 | 保留 Anthropic 新增的安全規則，在其基礎上擴展 |

### 追蹤版本變化

本倉庫的 `CHANGELOG.md` 記錄了每個版本的提示變動。建議在 Claude Code 更新前先查閱 changelog，了解哪些提示被修改：

```bash
# 查看特定提示的版本歷史
git log --oneline -- system-prompts/agent-prompt-explore.md
```

---

## 三種自訂路線比較

| 特性 | CLAUDE.md | Hooks | tweakcc |
|------|-----------|-------|---------|
| **修改層級** | 追加指令 | 攔截/觸發 | 替換底層提示 |
| **持久性** | 隨專案 | 隨使用者設定 | 隨安裝 |
| **複雜度** | 低 | 中 | 高 |
| **安全風險** | 低 | 中 | 高 |
| **維護成本** | 低 | 中 | 高（每次更新需重新 patch） |
| **適用場景** | 專案級編碼標準 | 自動化流程 | 深度行為修改 |
| **影響範圍** | 當前 session | 特定事件 | 所有 session |
| **團隊分享** | Git 提交 | 配置檔分享 | 配置檔 + script |
| **回滾難度** | 刪除檔案 | 移除 hook | `tweakcc restore` |
| **官方支援** | 官方推薦 | 官方功能 | 第三方工具 |

### 建議優先順序

1. **先用 CLAUDE.md**：大多數自訂需求（編碼標準、commit 風格、測試框架）都可以透過 CLAUDE.md 解決
2. **再用 Hooks**：需要自動化流程（自動格式化、自動測試、命令記錄）時使用
3. **最後用 tweakcc**：只有當需要從根本上修改 Claude Code 行為（如修改安全規則、替換提示邏輯）時才使用

### 組合使用範例

```
┌─────────────────────────────────────────────┐
│ tweakcc：修改底層提示                         │
│  - 強制 Conventional Commits 格式             │
│  - 加入內部工具的前綴偵測規則                  │
├─────────────────────────────────────────────┤
│ Hooks：自動化流程                             │
│  - PostToolUse: 程式碼修改後自動 lint          │
│  - PreToolUse: 記錄所有 bash 命令             │
├─────────────────────────────────────────────┤
│ CLAUDE.md：專案指令                           │
│  - 使用 TypeScript strict mode                │
│  - 測試框架用 vitest                          │
│  - 遵循團隊命名規範                           │
└─────────────────────────────────────────────┘
```
