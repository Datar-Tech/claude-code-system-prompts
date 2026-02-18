# Agent Teams 架構解析

> 基於 Claude Code v2.1.45 系統提示的深度分析。

## 目錄

- [概覽](#概覽)
- [與 Task 子代理的差異](#與-task-子代理的差異)
- [核心元件](#核心元件)
  - [TeamCreate](#teamcreate)
  - [SendMessage](#sendmessage)
  - [共享任務清單](#共享任務清單)
  - [TeamDelete](#teamdelete)
- [Delegate Mode](#delegate-mode)
- [代理類型選擇](#代理類型選擇)
- [訊息遞送機制](#訊息遞送機制)
- [Idle 狀態設計](#idle-狀態設計)
- [Plan Mode 審批整合](#plan-mode-審批整合)
- [演進歷史](#演進歷史)
- [設計特點與權衡](#設計特點與權衡)

---

## 概覽

Agent Teams（也稱 swarm）是 Claude Code 在 v2.1.16 引入的多代理協作系統。它將之前的「主代理 → 子代理」的單向指揮關係，擴展為一個具有**獨立身份、雙向通訊、共享任務清單、生命週期管理**的完整團隊協作框架。

架構圖：

```
                        使用者
                          │
                          ▼
  ┌─────────────────────────────────────────────┐
  │              Team Lead（團隊領導）              │
  │  - 與使用者直接互動                              │
  │  - 可進入 Delegate Mode（僅協調，不執行）        │
  │  - 或自己也做實作（Player-Coach 模式）           │
  │                                             │
  │  工具: TeamCreate, SendMessage, TaskCreate,     │
  │        TaskUpdate, TaskList, TeamDelete          │
  │  (Delegate Mode 下禁用 Bash/Read/Write/Edit)    │
  ├─────────────────────────────────────────────┤
  │                                             │
  │     ┌──────────┬──────────┬──────────┐     │
  │     │ Teammate │ Teammate │ Teammate │     │
  │     │ "researcher" │ "coder" │ "tester" │     │
  │     │ (Explore) │ (general) │ (general) │     │
  │     └─────┬────┴─────┬────┴─────┬────┘     │
  │           │          │          │           │
  │     ┌─────▼──────────▼──────────▼─────┐     │
  │     │    共享任務清單 (TaskList)         │     │
  │     │    ~/.claude/tasks/{team-name}/  │     │
  │     └─────────────────────────────────┘     │
  │                                             │
  │     ┌─────────────────────────────────┐     │
  │     │    團隊配置檔                      │     │
  │     │    ~/.claude/teams/{team-name}/  │     │
  │     │    config.json                   │     │
  │     └─────────────────────────────────┘     │
  └─────────────────────────────────────────────┘
```

---

## 與 Task 子代理的差異

| 特性 | Task 子代理 | Agent Teams |
|------|-----------|-------------|
| 通訊方向 | 單向（主 → 子 → 結果返回） | 雙向（任何成員可互相通訊） |
| 身份 | 匿名，用完即棄 | 具名（如 "researcher"、"tester"） |
| 生命週期 | 單次任務後銷毀 | 持續存活，接任務→完成→待命→再接 |
| 協調方式 | 主代理直接指派 | 共享任務清單 + 訊息系統 |
| 並行能力 | 有限（背景執行） | 原生多代理真並行 |

Agent Teams 建立在 Task 子代理之上：每個 teammate 本質上是一個帶有額外屬性（名稱、團隊歸屬、雙向通訊能力、idle/wake 循環）的增強型 Task 子代理。

---

## 核心元件

### TeamCreate

建立團隊時同時產生兩個持久化資源：

```json
{
  "team_name": "feature-auth",
  "description": "Implementing user authentication system"
}
```

產出：
- `~/.claude/teams/feature-auth/config.json` — 團隊配置（成員清單）
- `~/.claude/tasks/feature-auth/` — 共享任務目錄

**團隊配置檔結構：**

```json
{
  "members": [
    {
      "name": "team-lead",
      "agentId": "uuid-xxx",
      "agentType": "general-purpose"
    },
    {
      "name": "researcher",
      "agentId": "uuid-yyy",
      "agentType": "Explore"
    }
  ]
}
```

- `name`：人類可讀名稱，**所有通訊和任務指派都使用此名稱**
- `agentId`：唯一 ID，僅供系統內部參考
- `agentType`：代理類型，決定可用工具集

**觸發時機（來自系統提示 `tool-description-teammatetool.md`）：**
- 使用者明確要求使用 team / swarm / 多代理
- 使用者提到要代理「一起工作」、「協作」
- 任務複雜到需要並行處理（全端功能、邊重構邊維護測試等）
- 存疑時優先啟動團隊（`"When in doubt, prefer spawning a team"`）

### SendMessage

團隊內部的通訊工具，分四種訊息類型：

#### message（點對點訊息）

```json
{
  "type": "message",
  "recipient": "researcher",
  "content": "Found the auth module at src/auth/. Please investigate the token refresh logic.",
  "summary": "Auth module location for investigation"
}
```

- `recipient`：隊友名稱（必填）
- `content`：訊息內容（必填）
- `summary`：5-10 字摘要，在 UI 中顯示為預覽（必填）

#### broadcast（全體廣播）

```json
{
  "type": "broadcast",
  "content": "Critical: blocking bug found in shared utils, stop all work",
  "summary": "Critical blocking issue found"
}
```

**成本警告**：broadcast 的成本隨團隊人數線性增長（N 人 = N 次獨立訊息遞送）。僅用於真正需要全體關注的緊急情況。

#### shutdown_request / shutdown_response

請求隊友優雅關閉：

```json
{ "type": "shutdown_request", "recipient": "researcher", "content": "Task complete" }
```

隊友可以批准或拒絕：

```json
{ "type": "shutdown_response", "request_id": "abc-123", "approve": false,
  "content": "Still working on task #3, need 5 more minutes" }
```

#### plan_approval_response

審批隊友的實作計畫：

```json
{
  "type": "plan_approval_response",
  "request_id": "abc-123",
  "recipient": "researcher",
  "approve": false,
  "content": "Please add error handling for the API calls"
}
```

### 共享任務清單

所有成員共享同一個任務清單（`~/.claude/tasks/{team-name}/`）：

**任務屬性：**
- `subject`：簡短動作標題（祈使句）
- `description`：詳細描述，足以讓其他代理獨立完成
- `activeForm`：進行中顯示的文字（現在進行式）
- `status`：`pending` / `in_progress` / `completed`
- `owner`：擁有者（隊友名稱或 null）
- `blockedBy`：阻塞依賴（任務 ID 清單）
- `blocks`：此任務阻塞的其他任務

**隊友的任務認領流程**（來自 `tool-description-tasklist-teammate-workflow.md`）：

1. 完成當前任務後呼叫 `TaskList` 檢查可用工作
2. 尋找：`status: "pending"` + 無 owner + `blockedBy` 為空
3. **優先選擇 ID 最小的任務**（低 ID 通常為後續任務建立上下文）
4. 用 `TaskUpdate` 設定 `owner` 認領
5. 如果所有任務都被阻塞，通知 team lead 或幫助解除阻塞

### TeamDelete

清理團隊資源：

- 移除 `~/.claude/teams/{team-name}/`
- 移除 `~/.claude/tasks/{team-name}/`
- 清除 session 中的團隊上下文

**限制**：如果團隊還有活躍成員，`TeamDelete` 會失敗。必須先優雅關閉所有隊友。

---

## Delegate Mode

團隊領導可以進入「委派模式」——只能協調，不能執行：

| 狀態 | 可用工具 |
|------|---------|
| Delegate Mode **開啟** | TeamCreate, SendMessage, TaskCreate, TaskGet, TaskUpdate, TaskList |
| Delegate Mode **關閉** | 所有工具（Bash, Read, Write, Edit, Glob, Grep, WebFetch...） |

來自 `system-reminder-delegate-mode-prompt.md`：

> *「You CANNOT use any other tools (Bash, Read, Write, Edit, etc.) until you exit delegate mode. Focus on coordinating work by creating tasks, assigning them to teammates, and monitoring progress.」*

退出 delegate mode 後恢復所有工具存取。

---

## 代理類型選擇

系統提示（`tool-description-teammatetool.md`）要求根據任務需求選擇正確的代理類型：

```
               需要修改檔案嗎？
                ┌─────┴─────┐
                是           否
                ▼            ▼
          general-purpose   需要設計方案嗎？
                            ┌──┴──┐
                            是    否
                            ▼     ▼
                          Plan   Explore
```

| 代理類型 | 能力 | 適合的任務 |
|---------|------|-----------|
| **Explore** | 唯讀（Glob, Grep, Read, Bash 唯讀） | 搜尋、研究、程式碼探索 |
| **Plan** | 唯讀，結構化方案輸出 | 架構設計、方案規劃 |
| **general-purpose** | 完整工具集 | 寫程式碼、修改檔案、執行命令 |
| **Bash** | shell 執行 | 命令執行 |
| **自訂代理** (`.claude/agents/`) | 依定義而異 | 專案特定角色 |

關鍵規則：

> *「Read-only agents (e.g., Explore, Plan) cannot edit or write files. Only assign them research, search, or planning tasks. **Never assign them implementation work.**」*

---

## 訊息遞送機制

```
Teammate A 發送訊息給 Teammate B
        │
        ▼
  B 是否正在執行中 (mid-turn)?
        ├── 是 → 訊息排入佇列 → B 的 turn 結束後自動遞送
        └── 否 (idle) → 直接遞送 → B 被喚醒處理訊息

  UI 顯示通知：「[sender name] 的訊息等待中」
```

來自 `tool-description-teammatetool.md`：

> *「Messages from teammates are automatically delivered to you. You do NOT need to manually check your inbox.」*

**Peer DM 可見性**：當隊友 A 發 DM 給隊友 B 時，Team Lead 會在 A 的 idle 通知中看到一個簡短摘要，提供監督權但不干擾細節。

---

## Idle 狀態設計

系統提示花了大量篇幅解釋 idle 狀態，因為這是最容易被誤解的概念：

| 事實 | 說明 |
|------|------|
| idle 不代表「完成」 | 只是「等待輸入」 |
| idle 不代表「離線」 | idle 的隊友可以接收訊息，收到後被喚醒 |
| idle 通知是自動的 | 每次 turn 結束系統自動發送，不需要反應 |
| idle 是正常狀態 | 發送訊息後進入 idle 是標準流程 |

> *「Be patient with idle teammates! Don't comment on their idleness until it actually impacts your work.」*

---

## Plan Mode 審批整合

隊友可被設定為 `plan_mode_required`，實作前必須先提交計畫給 lead 審批：

```
1. Teammate 進入 Plan Mode
2. 探索程式碼 → 設計方案 → 寫入計畫檔案
3. 呼叫 ExitPlanMode → 系統發送 plan_approval_request 給 lead
4. Lead 審查計畫
5a. 批准 → teammate 自動退出 Plan Mode → 開始實作
5b. 拒絕（附帶反饋）→ teammate 修改計畫 → 重新提交
```

---

## 演進歷史

Agent Teams 在短短幾個版本中經歷了快速的架構迭代：

| 版本 | 變化 | Token 影響 |
|------|------|-----------|
| **v2.1.16** | 首次引入：TeammateTool、SendMessage、Delegate Mode、Team Coordination、Team Shutdown、TaskCreate、5-phase Plan Mode | **+7,114 tks** |
| v2.1.19 | TaskUpdate 大幅擴展：任務擁有權、團隊協調、認領機制 | |
| v2.1.20 | 加入「優先低 ID 任務」規則 | |
| v2.1.21 | 短暫移除 Delegate Mode、Team Coordination（後來恢復） | |
| v2.1.23 | 加入團隊發現/加入操作（discoverTeams, requestJoin） | |
| v2.1.26 | 加入 peer DM 可見性 | |
| v2.1.30 | SendMessage 扁平化協議（shutdown_request/response、plan_approval_response） | |
| **v2.1.32** | 主要重構：移除 discoverTeams/requestJoin、加入 "When to Use" 和 "Choosing Agent Types" 指南、簡化通訊規則 | **+2,323 tks** |
| **v2.1.33** | 拆分 TeamDelete 為獨立工具、移除 AGENT_TEAM_CHECK 付費方案限制 | -1,086 tks |
| v2.1.38 | TaskList 加入 teammate workflow 條件區塊 | |

趨勢觀察：
- 整體方向是**簡化協議**（從複雜操作參數到扁平化訊息類型）
- 增強**隊友自治性**（自主認領而非領導指派）
- v2.1.33 移除付費方案限制，Agent Teams 從付費功能轉為通用功能

---

## 設計特點與權衡

**優點：**

1. **真並行**：多個代理同時在不同檔案/模組上工作
2. **自組織**：隊友自主認領任務，不需要領導微管理
3. **可見性**：peer DM 摘要讓領導掌握全局
4. **優雅關機**：隊友可以拒絕關機，防止工作中斷

**權衡：**

1. **成本**：每個 teammate 是獨立的 API session，token 消耗倍增
2. **協調開銷**：訊息傳遞本身消耗 API 資源，broadcast 特別昂貴
3. **衝突風險**：多個代理同時修改同一檔案可能產生衝突（系統提示未明確處理此問題）
4. **Delegate Mode 取捨**：領導放棄工具存取換取專注協調，但有時需要快速檢查檔案而不得不退出
