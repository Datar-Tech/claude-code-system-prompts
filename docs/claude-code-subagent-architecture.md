# Claude Code 子代理（Sub-agent）架構深度解析

> 基於 Claude Code v2.1.45 系統提示的深度分析。

## 目錄

- [架構總覽](#架構總覽)
- [子代理分類](#子代理分類)
  - [使用者可見的核心子代理](#a-使用者可見的核心子代理)
  - [隱形功能子代理](#b-隱形功能子代理)
- [子代理調度機制](#子代理調度機制)
- [子代理間的資訊流動](#子代理間的資訊流動)
- [安全隔離設計](#安全隔離設計)
- [自訂子代理](#自訂子代理)

---

## 架構總覽

Claude Code 的子代理系統是一個多層級、角色分離的代理生態系統。主代理（Main Agent）透過 `Task` 工具啟動子代理，每個子代理擁有獨立的 context window、專屬的系統提示、受限的工具集，以及明確的職責邊界。

```
┌──────────────────────────────────────────────────────┐
│                  主代理 (Main Agent)                    │
│  - 完整的工具集 (Read/Edit/Write/Bash/WebFetch...)      │
│  - 使用者對話的完整上下文                                 │
│  - 透過 Task tool 調度子代理                             │
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────┐ │
│  │ Explore  │  │  Plan    │  │  Bash (Command     │ │
│  │ 唯讀探索  │  │ 唯讀規劃  │  │  Execution)        │ │
│  └──────────┘  └──────────┘  └────────────────────┘ │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │          通用子代理 (general-purpose)           │    │
│  │  搜尋、分析、研究、寫程式碼                       │    │
│  └──────────────────────────────────────────────┘    │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────┐ │
│  │ 隱形代理  │  │ 隱形代理  │  │ 隱形代理            │ │
│  │ 摘要/情緒 │  │ 標題生成  │  │ 命令前綴偵測/安全    │ │
│  └──────────┘  └──────────┘  └────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

---

## 子代理分類

### A. 使用者可見的核心子代理

透過 Task tool 啟動，使用者可直接觀察其運作：

| 子代理 | 角色 | 工具權限 | 核心限制 |
|--------|------|----------|---------|
| **Explore** | 程式碼庫快速搜尋探索 | Glob, Grep, Read, Bash(唯讀) | **嚴格唯讀** |
| **Plan** | 軟體架構師，設計實作方案 | Glob, Grep, Read, Bash(唯讀) | **嚴格唯讀** |
| **Bash** | 命令執行專家 | Bash | 遵循 git 安全協議 |
| **general-purpose** | 通用搜尋/分析/寫程式碼 | 全工具集 | 無特殊限制 |

#### Explore 子代理

來源：`agent-prompt-explore.md`（516 tks）

- 被定義為「快速代理」（fast agent），要求盡快返回結果
- 強制使用並行工具呼叫提高效率
- 禁止清單極其詳盡：不可 `mkdir`、`touch`、`rm`、`cp`、`mv`、`git add`、`git commit`、`npm install`、`pip install`，甚至不可使用重定向運算符
- 最終報告以純文字返回，不可建立檔案

> *「NOTE: You are meant to be a fast agent that returns output as quickly as possible.」*

#### Plan 子代理

來源：`agent-prompt-plan-mode-enhanced.md`（633 tks）

- 結構化流程：理解需求 → 探索程式碼 → 設計方案 → 詳細計畫
- 輸出必須包含「Critical Files for Implementation」清單（3-5 個關鍵檔案及原因）
- 可接收 Explore 代理的探索結果作為輸入

#### 何時用子代理 vs 直接呼叫工具

來自 `system-prompt-conditional-delegate-codebase-exploration.md`：

| 場景 | 策略 |
|------|------|
| 已知檔案路徑 | 直接用 `Read` |
| 搜尋特定類別定義 | 直接用 `Glob` |
| 在 2-3 個已知檔案中搜尋 | 直接用 `Read` |
| 廣泛的程式碼庫探索、需要 >3 次查詢 | 用 `Explore` 子代理 |
| 需要架構規劃 | 用 `Plan` 子代理 |

### B. 隱形功能子代理

在背後自動運作，使用者通常不直接看到。

#### 對話管理類

| 子代理 | Token 數 | 功能 |
|--------|---------|------|
| **Conversation summarization** | 1121 | context window 接近上限時產生 9 章節結構化摘要 |
| **Recent message summarization** | 720 | 只摘要最近訊息，保留早期上下文 |
| **Context compaction** | 278 | SDK 用的壓縮摘要 |

#### 使用者分析類

| 子代理 | Token 數 | 功能 |
|--------|---------|------|
| **User sentiment analysis** | 205 | 分析使用者挫折感和 PR 建立請求 |
| **Prompt suggestion generator** | 283/296 | 推薦後續提示建議 |

情緒分析子代理輸出 `<frustrated>true/false</frustrated>` 和 `<pr_request>true/false</pr_request>`。

#### 安全與權限類

| 子代理 | Token 數 | 功能 |
|--------|---------|------|
| **Bash command prefix detection** | 823 | 分析 bash 命令、提取前綴、偵測命令注入 |
| **Hook condition evaluator** | 78 | 評估 hook 條件，返回 `{"ok": true/false}` |

命令前綴偵測是安全關鍵代理，詳見 [claude-code-security.md](./claude-code-security.md)。

#### 文件管理類

| 子代理 | Token 數 | 功能 |
|--------|---------|------|
| **CLAUDE.md creation** | 384 | 分析程式碼庫，生成 CLAUDE.md |
| **Session memory update** | 756 | 更新 session 筆記，保留模板結構 |
| **Update Magic Docs** | 718 | 更新魔法文件，強調簡潔高信號量 |
| **Session title and branch generation** | 307 | 生成標題（≤6 詞）和分支名（`claude/xxx`） |

#### Slash Command 子代理

| 子代理 | Token 數 | 功能 |
|--------|---------|------|
| **/review-pr** | 211 | 用 `gh pr view` + `gh pr diff` 做 PR 審查 |
| **/security-review** | 2610 | 三階段安全審查，最大的子代理提示 |
| **/pr-comments** | 402 | 擷取和顯示 PR 評論 |

---

## 子代理調度機制

### 啟動方式

主代理透過 `Task` 工具啟動子代理，需指定：

```
- subagent_type: "Explore" | "Plan" | "Bash" | "general-purpose" | ...
- prompt: 詳細的任務描述
- description: 3-5 字摘要
- run_in_background: true/false（可選背景執行）
- resume: agent_id（可選恢復之前的代理）
```

### 並行調度

**計畫模式下的 5 階段並行**（來自 `system-reminder-plan-mode-is-active-5-phase.md`）：

1. **Phase 1**：啟動最多 N 個 Explore 子代理**並行**探索
2. **Phase 2**：啟動最多 N 個 Plan 子代理**並行**設計方案（不同角度）
3. **Phase 3**：主代理審閱所有方案
4. **Phase 4**：寫最終計畫
5. **Phase 5**：退出計畫模式

**安全審查的多層並行**（來自 `agent-prompt-security-review-slash-command.md`）：

1. 一個子代理掃描所有漏洞
2. 為每個漏洞**並行**啟動獨立子代理做誤報過濾
3. 主代理篩選 confidence ≥ 8 的結果

---

## 子代理間的資訊流動

```
使用者訊息
    │
    ▼
主代理 ────────────────────────────────┐
    │                                  │
    ├─→ Explore 子代理 ──→ 返回搜尋結果   │
    │                                  │
    ├─→ Plan 子代理 ←── 接收 Explore 結果 │
    │        └──→ 返回設計方案            │
    │                                  │
    ├─→ 情緒分析子代理 ──→ frustrated?   │
    │                                  │
    ├─→ 命令前綴偵測子代理 ──→ 安全/注入?  │
    │                                  │
    ├─→ 摘要子代理 ──→ 壓縮上下文         │
    │                                  │
    └─→ 主代理整合所有結果，回覆使用者    ◄┘
```

關鍵設計：
- 子代理的輸出**對使用者不可見**——主代理必須在回覆中總結結果
- 子代理可以被「恢復」（resume），保留完整的先前上下文
- 背景子代理的輸出寫入檔案，主代理可隨時用 `Read` 檢查進度
- 每個子代理的 **cwd 在 bash 呼叫間會被重設**，必須使用絕對路徑

---

## 安全隔離設計

| 層級 | 機制 |
|------|------|
| **工具隔離** | Explore/Plan 代理無法存取 Edit/Write 工具，嘗試會失敗 |
| **唯讀強制** | 系統提示中用 `=== CRITICAL: READ-ONLY MODE ===` 強調 |
| **命令注入偵測** | 專用子代理分析每條 bash 命令（詳見安全文件） |
| **前綴白名單** | 使用者允許的命令以前綴為單位，注入偵測可繞過白名單 |
| **情緒監控** | 隱形代理偵測使用者挫折感，調整互動策略 |
| **結果不可見** | 子代理結果不直接暴露給使用者，由主代理過濾後總結 |

---

## 自訂子代理

Claude Code 允許透過 **Agent creation architect**（`agent-prompt-agent-creation-architect.md`，1110 tks）建立自訂代理。

這個「代理建立代理」會：

1. 提取使用者的核心意圖
2. 設計專家角色身份
3. 撰寫完整的系統提示
4. 建立觸發條件（含範例）
5. 輸出 JSON：`{identifier, whenToUse, systemPrompt}`

自訂代理會考慮 CLAUDE.md 上下文，確保與既有編碼標準對齊。

---

## 設計原則總結

1. **最小權限**：每個代理只有完成任務所需的最少工具
2. **關注點分離**：探索、規劃、執行、安全檢查各有專責
3. **並行優先**：盡可能並行啟動多個獨立代理提升效率
4. **上下文保護**：用子代理隔離大量搜尋結果，避免污染主 context window
5. **安全縱深**：多層安全機制從命令注入偵測到工具隔離全面覆蓋
6. **可擴展性**：使用者可透過 agent architect 自訂新代理
