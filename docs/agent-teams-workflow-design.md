# Agent Teams 工作流設計指南

> 基於 Claude Code v2.1.45 系統提示的深度分析，提煉出有效設計 Agent Teams 工作流的原則與模式。

## 目錄

- [核心設計原則](#核心設計原則)
- [何時該用 / 不該用 Agent Teams](#何時該用--不該用-agent-teams)
- [任務分解策略](#任務分解策略)
  - [按檔案邊界切割](#原則-1按檔案模組邊界切割)
  - [利用依賴圖](#原則-2利用-blockedby-建立依賴圖)
  - [任務描述要充分](#原則-3任務描述要足夠讓另一個代理獨立完成)
- [代理角色設計模板](#代理角色設計模板)
- [領導模式選擇](#領導模式選擇)
- [依賴管理與解除阻塞](#依賴管理與解除阻塞)
- [與 Skills 系統的整合](#與-skills-系統的整合)
- [完整範例](#完整範例全端認證功能)
- [常見陷阱](#常見陷阱)

---

## 核心設計原則

從系統提示中可以提煉出 Agent Teams 的設計哲學：

```
有效的 Agent Team ≠ 越多代理越好
有效的 Agent Team = 最少的代理 × 最清晰的分工 × 最少的通訊
```

系統提示中的關鍵線索：

> *「When in doubt about whether a task warrants a team, prefer spawning a team.」*
> —— 但同時——
> *「Broadcasting is expensive. Each broadcast sends a separate message to every teammate, which means N teammates = N separate message deliveries.」*

含義：**低門檻啟動團隊，但高標準控制團隊規模和通訊量。**

---

## 何時該用 / 不該用 Agent Teams

### 適合 Agent Teams

| 場景 | 為什麼適合 | 團隊結構建議 |
|------|-----------|-------------|
| 全端功能開發 | 前後端可獨立且並行 | frontend-dev + backend-dev + tester |
| 大型重構 + 測試維護 | 重構和測試修復可並行 | refactorer + test-fixer |
| 多步驟流程（研究→規劃→實作） | 研究產出直接傳遞給實作 | researcher(Explore) + implementer(general) |
| 多模組相關的 bug 修復 | 分頭調查不同模組 | investigator-a + investigator-b |

### 不適合 Agent Teams（直接用 Task 子代理或直接做）

| 場景 | 為什麼不適合 | 替代方案 |
|------|-------------|---------|
| 單一檔案修改 | 協調成本遠超任務本身 | 直接操作 |
| 純探索性問題 | 不需要持久生命週期 | Explore 子代理 |
| 高度序列的任務鏈 | 每步依賴前一步，無法並行 | Plan Mode + 直接執行 |
| 需要頻繁使用者確認 | 團隊內部無法觸發使用者確認 | 直接互動模式 |

---

## 任務分解策略

任務分解是整個工作流成敗的關鍵。

### 原則 1：按檔案/模組邊界切割

避免多個代理同時修改同一檔案。

```
❌ 錯誤切割：
   Task A: "實作所有 API endpoints"
   Task B: "為所有 API endpoints 加上驗證"
   → 兩個代理會修改同一批檔案，產生衝突

✅ 正確切割：
   Task A: "實作並驗證 /users endpoints"（src/routes/users.ts）
   Task B: "實作並驗證 /products endpoints"（src/routes/products.ts）
   → 各自負責獨立檔案
```

### 原則 2：利用 blockedBy 建立依賴圖

```
TaskCreate: "Research existing auth patterns"         → ID: 1
TaskCreate: "Design auth database schema"             → ID: 2, blockedBy: [1]
TaskCreate: "Implement JWT middleware"                 → ID: 3, blockedBy: [2]
TaskCreate: "Create login API endpoint"               → ID: 4, blockedBy: [2, 3]
TaskCreate: "Build login React component"             → ID: 5, blockedBy: [2]
TaskCreate: "Write E2E auth tests"                    → ID: 6, blockedBy: [4, 5]
```

視覺化依賴圖：

```
  [1] Research
      │
      ▼
  [2] Schema Design
    ┌──┴───┬────────┐
    ▼      ▼        ▼
  [3] JWT [5] React  │
    │       │        │
    ▼       │        │
  [4] API ──┘        │
    │                │
    ▼                │
  [6] E2E Tests ◄────┘
```

隊友會自動遵循此依賴（來自 `tool-description-tasklist-teammate-workflow.md`）：

> *「Look for tasks with status 'pending', no owner, and empty blockedBy」*
> *「Prefer tasks in ID order (lowest ID first)」*

### 原則 3：任務描述要足夠讓另一個代理獨立完成

來自 `tool-description-taskcreate.md`：

> *「Include enough detail in the description for another agent to understand and complete the task」*

```
❌ 模糊任務:
   subject: "Fix the auth bug"
   description: "There's a bug in auth"

✅ 明確任務:
   subject: "Fix JWT token refresh race condition"
   description: "In src/auth/tokenRefresh.ts, when two API calls
   trigger refresh simultaneously, the second call gets a stale token.
   Fix by adding a mutex/lock pattern. Reference the existing lock
   implementation in src/utils/asyncLock.ts. Verify by running
   npm test -- --grep 'token refresh'"
```

好的任務描述應包含：
- 具體的檔案路徑
- 問題的根因或需求的明確描述
- 可參考的現有程式碼模式
- 驗證方式（測試命令等）

---

## 代理角色設計模板

### 模板 A：最小團隊（2 人）

```
適用：中等複雜度的功能開發

team-lead: 自己做實作 + 協調
  └── researcher (Explore): 探索程式碼庫 → 產出交給 lead 實作
```

### 模板 B：前後端分離團隊（3 人）

```
適用：全端功能

team-lead: 協調 + 整合測試
  ├── frontend-dev (general-purpose): UI 元件 + 前端邏輯
  └── backend-dev (general-purpose): API + 資料庫
```

### 模板 C：研究-實作-驗證團隊（3-4 人）

```
適用：需要大量探索的新功能

team-lead: 協調 + 最終整合
  ├── researcher (Explore): 程式碼庫探索 + 模式發現
  ├── implementer (general-purpose): 核心實作
  └── tester (general-purpose): 測試撰寫 + 驗證
```

### 模板 D：並行重構團隊（N 人）

```
適用：大型重構，多個獨立模組

team-lead: 協調 + 依賴管理
  ├── refactor-auth (general-purpose): 重構 auth 模組
  ├── refactor-api (general-purpose): 重構 API 層
  └── refactor-db (general-purpose): 重構資料庫層
```

---

## 領導模式選擇

### 選項 A：Delegate Mode（純協調者）

進入 Delegate Mode 後只能使用協調工具，禁用所有執行工具。

**適用場景：**
- 所有實作工作都可以交給隊友
- 任務邊界清晰，不需要 lead 即時檢查程式碼
- 較大的團隊（3+ 隊友），lead 的注意力應集中在協調

**風險：** 需要快速檢查檔案來做決策時，必須退出 delegate mode。

### 選項 B：Player-Coach（邊做邊管）

不進入 Delegate Mode，lead 保留所有工具，自己也接任務。

**適用場景：**
- 小團隊（1-2 隊友）
- Lead 負責最關鍵/最複雜的部分
- 需要頻繁讀取程式碼來做協調決策

**風險：** Lead 做自己的任務時可能忽略隊友的訊息。

---

## 依賴管理與解除阻塞

### 利用任務系統管理依賴

```
Phase 1: 獨立任務（可立即並行）
  [1] "Explore existing database schema"           blockedBy: []
  [2] "Review current API patterns"                blockedBy: []

Phase 2: 依賴 Phase 1
  [3] "Design new user table migration"            blockedBy: [1]
  [4] "Implement user CRUD API"                    blockedBy: [2, 3]
  [5] "Create user settings React page"            blockedBy: [2]

Phase 3: 依賴 Phase 2
  [6] "Write integration tests"                    blockedBy: [4, 5]
  [7] "Update API documentation"                   blockedBy: [4]
```

### 處理阻塞的策略

來自系統提示：

> *「If all available tasks are blocked, notify the team lead or help resolve blocking tasks」*

建議：
- 每個 Phase 至少有一個不被阻塞的任務，讓隊友不會空閒
- 如果預見到瓶頸，提前拆分關鍵路徑上的任務
- 讓被阻塞的隊友可以幫助「解除阻塞任務」（例如寫測試的人先寫測試框架和 fixtures，等 API 完成再填入具體測試案例）

---

## 與 Skills 系統的整合

### SKILL.md 中的 Execution 標註

Skillify 系統（`/skillify`）可以把成功的 Agent Teams 會話封裝成可重複的技能。來自 `system-prompt-skillify-current-session.md`：

```markdown
### 3a. Implement frontend changes
What to do...
**Execution**: Teammate        ← 標記為用 Teammate 執行（支持真並行）
**Artifacts**: Component files modified
**Success criteria**: All components render without console errors

### 3b. Implement backend changes
What to do...
**Execution**: Teammate        ← 可與 3a 並行
**Artifacts**: API endpoints created
**Success criteria**: All endpoints return correct status codes
```

四種 Execution 類型：
- `Direct`（預設）：Team Lead 直接執行
- `Task agent`：啟動簡單子代理
- `Teammate`：使用 Agent Teams 的真並行代理
- `[human]`：需要使用者手動操作

### 自訂代理定義

來自 `tool-description-teammatetool.md`：

> *「Custom agents defined in `.claude/agents/` may have their own tool restrictions.」*

可以在專案中建立專屬的代理定義，然後在 Agent Teams 中使用：

```
.claude/agents/
├── security-reviewer.md    ← 專注安全審查
├── db-migrator.md          ← 專注資料庫遷移
└── api-tester.md           ← 專注 API 測試
```

---

## 完整範例：全端認證功能

```
使用者: 「幫我實作完整的 JWT 認證系統，包含
        登入/註冊頁面、API endpoints、資料庫遷移、和測試」

Step 1: Team Lead 建立團隊
────────────────────────────
TeamCreate { team_name: "auth-system" }

Step 2: 建立結構化任務清單
────────────────────────────
[1]  "Explore existing auth patterns and DB schema"    blockedBy: []
[2]  "Explore existing React form patterns"            blockedBy: []
[3]  "Create users table migration"                    blockedBy: [1]
[4]  "Implement JWT auth middleware"                   blockedBy: [1]
[5]  "Create /auth/login and /auth/register APIs"      blockedBy: [3, 4]
[6]  "Build LoginForm React component"                 blockedBy: [2]
[7]  "Build RegisterForm React component"              blockedBy: [2]
[8]  "Connect frontend forms to auth APIs"             blockedBy: [5, 6, 7]
[9]  "Write auth API integration tests"                blockedBy: [5]
[10] "Write auth E2E tests with Playwright"            blockedBy: [8]

Step 3: 啟動隊友（3 人）
────────────────────────────
Task { name: "backend-dev",  subagent_type: "general-purpose",
       team_name: "auth-system",
       prompt: "You are implementing the backend auth system.
               Check TaskList for your tasks. Start with
               exploration tasks, then implement endpoints." }

Task { name: "frontend-dev", subagent_type: "general-purpose",
       team_name: "auth-system",
       prompt: "You are implementing the frontend auth UI.
               Check TaskList for your tasks. Start with
               exploring existing React patterns." }

Task { name: "test-writer",  subagent_type: "general-purpose",
       team_name: "auth-system",
       prompt: "You are responsible for all auth tests.
               Check TaskList for available testing tasks.
               Start when API and UI tasks unblock." }

Step 4: 自動化執行（並行時間線）
────────────────────────────
  T0 ─── backend-dev 認領 [1] → 探索 DB schema
  T0 ─── frontend-dev 認領 [2] → 探索 React patterns
  T0 ─── test-writer → 無可用任務 → idle

  T1 ─── backend-dev 完成 [1] → [3],[4] 解除阻塞 → 認領 [3]
  T1 ─── frontend-dev 完成 [2] → [6],[7] 解除阻塞 → 認領 [6]
  T1 ─── test-writer → 仍無可用任務 → idle

  T2 ─── backend-dev 完成 [3] → 認領 [4] (JWT middleware)
  T2 ─── frontend-dev 完成 [6] → 認領 [7] (RegisterForm)

  T3 ─── backend-dev 完成 [4] → [5] 解除阻塞 → 認領 [5]
  T3 ─── frontend-dev 完成 [7] → 無可用任務 → idle

  T4 ─── backend-dev 完成 [5] → [8],[9] 解除阻塞
  T4 ─── test-writer 認領 [9] → 開始寫整合測試
  T4 ─── frontend-dev 認領 [8] → 連接前端到 API

  T5 ─── test-writer 完成 [9] → idle
  T5 ─── frontend-dev 完成 [8] → [10] 解除阻塞 → idle

  T6 ─── test-writer 認領 [10] → 執行 E2E 測試

  T7 ─── 所有任務完成

Step 5: Team Lead 優雅關閉
────────────────────────────
  SendMessage(shutdown_request) → backend-dev   → approve
  SendMessage(shutdown_request) → frontend-dev  → approve
  SendMessage(shutdown_request) → test-writer   → approve
  TeamDelete → 清理
  → 向使用者報告最終結果
```

---

## 常見陷阱

| 陷阱 | 原因 | 解法 |
|------|------|------|
| 多個代理修改同一檔案 | 造成衝突或覆蓋 | 按檔案/模組邊界劃分任務 |
| 任務描述不夠具體 | 代理無法獨立完成 | 包含檔案路徑、函式名、驗證命令 |
| 全部用 broadcast 溝通 | 成本倍增，資訊噪音 | 預設 message，broadcast 僅用於緊急 |
| Lead 微管理 idle 狀態 | 浪費 lead 的 turn | 信任系統，idle 是正常狀態 |
| 沒有設定 blockedBy | 代理可能跳過前置任務 | 所有有依賴的任務都要標記 |
| 純序列依賴鏈 | 和單代理沒區別，白浪費 | 重新設計任務以增加並行度 |
| 啟動太多隊友 | token 成本線性增長 | 以「最大同時可並行任務數」決定人數 |
| 用文字輸出和隊友「溝通」 | 隊友看不到純文字 | 必須用 SendMessage |
| 忘記 TaskUpdate(completed) | 依賴任務無法解除阻塞 | 完成即標記，不要批次更新 |
| 在有活躍成員時 TeamDelete | 會失敗 | 先全部 shutdown → 再 delete |
