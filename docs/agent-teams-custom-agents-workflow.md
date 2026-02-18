# Agent Teams + 自訂代理完整工作流

> 基於 Claude Code v2.1.45 系統提示的深度分析。

## 目錄

- [自訂代理基礎](#自訂代理基礎)
  - [定義方式](#定義方式)
  - [代理定義結構](#代理定義結構)
  - [建立方法](#建立方法)
- [在 Agent Teams 中使用自訂代理](#在-agent-teams-中使用自訂代理)
  - [代理類型選擇](#代理類型選擇)
  - [團隊配置](#團隊配置)
- [完整工作流範例：電商結帳功能](#完整工作流範例電商結帳功能)
  - [Phase 0：準備自訂代理](#phase-0準備自訂代理)
  - [Phase 1：使用者啟動任務](#phase-1使用者啟動任務)
  - [Phase 2：建立團隊與任務](#phase-2建立團隊與任務)
  - [Phase 3：團隊協作執行](#phase-3團隊協作執行)
  - [Phase 4：處理安全審查結果](#phase-4處理安全審查結果)
  - [Phase 5：優雅關閉](#phase-5優雅關閉)
- [自訂代理設計要點](#自訂代理設計要點)
- [工具集對照表](#工具集對照表)

---

## 自訂代理基礎

### 定義方式

自訂代理以 **markdown 檔案**的形式存放在專案根目錄的 `.claude/agents/` 中。每個檔案定義一個代理，Claude Code 在執行時會動態發現這些定義，讓它們與內建類型（Explore、Plan、general-purpose、Bash）一起作為可選的代理類型。

```
my-project/
├── .claude/
│   └── agents/
│       ├── security-reviewer.md
│       ├── db-migrator.md
│       └── api-tester.md
├── src/
├── CLAUDE.md
└── package.json
```

將 `.claude/agents/` 提交到 Git 後，全團隊共用相同的代理定義。

### 代理定義結構

每個 markdown 檔案包含一個 JSON 物件，有三個必要欄位：

```json
{
  "identifier": "kebab-case-name",
  "whenToUse": "Use this agent when... [含觸發條件與範例]",
  "systemPrompt": "You are a... [完整系統提示]"
}
```

| 欄位 | 說明 |
|------|------|
| `identifier` | 唯一識別碼，小寫字母 + 數字 + 連字號，2-4 個詞（如 `security-reviewer`、`db-migrator`） |
| `whenToUse` | 以「Use this agent when...」開頭的觸發條件描述，含使用範例 |
| `systemPrompt` | 完整的系統提示，以第二人稱撰寫（「You are...」、「You will...」），定義行為邊界、方法論、輸出格式 |

### 建立方法

**方式 A：手動建立**

直接在 `.claude/agents/` 中建立 markdown 檔案。

**方式 B：Agent Creation Architect 自動建立**

告訴 Claude Code 你想要什麼代理，它會呼叫內建的 Agent Creation Architect（`agent-prompt-agent-creation-architect.md`，1110 tks）自動設計：

```
> 幫我建立一個專門做資料庫遷移的代理
```

Agent Creation Architect 的流程：
1. 提取使用者的核心意圖
2. 設計專家角色身份
3. 撰寫完整的系統提示（含行為邊界、方法論、邊界案例處理、品質控制）
4. 建立識別碼
5. 輸出 JSON 並存到 `.claude/agents/`

它會讀取 CLAUDE.md 確保自訂代理符合專案規範。

---

## 在 Agent Teams 中使用自訂代理

### 代理類型選擇

建立團隊隊友時，用自訂代理的 `identifier` 作為 `subagent_type`：

```
需要什麼能力？
    │
    ├── 只需搜尋/研究 → Explore（內建，唯讀）
    ├── 只需設計方案 → Plan（內建，唯讀）
    ├── 需要通用開發 → general-purpose（內建，全工具）
    ├── 需要專業安全審查 → checkout-security-auditor（自訂）
    └── 需要專業測試撰寫 → e2e-test-writer（自訂）
```

來自 `tool-description-teammatetool.md` 的關鍵規則：

> *「Custom agents defined in `.claude/agents/` may have their own tool restrictions. Check their descriptions to understand what they can and cannot do.」*

> *「Read-only agents cannot edit or write files. Only assign them research, search, or planning tasks. Never assign them implementation work.」*

### 團隊配置

團隊配置檔（`~/.claude/teams/{team-name}/config.json`）中記錄每個隊友的代理類型：

```json
{
  "members": [
    { "name": "team-lead", "agentId": "uuid-1", "agentType": "general-purpose" },
    { "name": "researcher", "agentId": "uuid-2", "agentType": "Explore" },
    { "name": "sec-auditor", "agentId": "uuid-3", "agentType": "checkout-security-auditor" }
  ]
}
```

自訂代理在團隊中的行為與內建代理完全一致——透過 SendMessage 通訊、透過 TaskList 認領工作、遵循 idle/wake 生命週期。

---

## 完整工作流範例：電商結帳功能

以下展示一個完整的端到端流程——**為電商平台實作購物車結帳功能**，結合 Stripe 付款整合、安全審查和 E2E 測試。

### Phase 0：準備自訂代理

建立兩個專案特定的自訂代理：

#### `.claude/agents/checkout-security-auditor.md`

```json
{
  "identifier": "checkout-security-auditor",
  "whenToUse": "Use this agent when checkout or payment-related code changes need security review. Examples: <example>Context: A teammate finished implementing the payment API endpoint. user: \"Payment endpoint is ready for review\" assistant: \"Let me launch the checkout-security-auditor to verify payment security\" <commentary>Payment code was written, so use the security auditor to check for vulnerabilities before merging.</commentary></example>",
  "systemPrompt": "You are a senior payment security engineer with expertise in PCI-DSS compliance, OWASP payment vulnerabilities, and secure transaction processing.\n\nYour responsibilities:\n1. Review all payment-related code for security vulnerabilities\n2. Verify PCI-DSS compliance patterns (no plaintext card storage, proper tokenization)\n3. Check for injection attacks in transaction queries\n4. Validate input sanitization on price/quantity fields\n5. Ensure proper authentication/authorization on payment endpoints\n6. Check for race conditions in inventory/payment flows\n\nOutput format:\n- CRITICAL: Issues that must be fixed before merge\n- WARNING: Issues that should be addressed\n- INFO: Suggestions for improvement\n\nYou have read-only access. Report findings to the team lead via SendMessage."
}
```

#### `.claude/agents/e2e-test-writer.md`

```json
{
  "identifier": "e2e-test-writer",
  "whenToUse": "Use this agent when a feature implementation is complete and needs end-to-end test coverage. <example>Context: The checkout flow frontend and backend are both implemented. user: \"Checkout feature is ready for testing\" assistant: \"Launching the e2e-test-writer to create comprehensive test coverage\" <commentary>Feature implementation is done, use the e2e test writer to create tests.</commentary></example>",
  "systemPrompt": "You are a senior QA engineer specializing in end-to-end testing with Playwright.\n\nYour workflow:\n1. Read the implemented feature code to understand all user flows\n2. Identify happy paths, edge cases, and error scenarios\n3. Write Playwright tests covering:\n   - Complete checkout flow (add to cart → payment → confirmation)\n   - Error handling (invalid card, network timeout, out of stock)\n   - Edge cases (concurrent purchases, price changes during checkout)\n4. Run the tests and fix any failures\n\nConventions:\n- Test files go in tests/e2e/\n- Use Page Object pattern\n- Each test must be independent and idempotent"
}
```

最終專案結構：

```
my-ecommerce/
├── .claude/
│   └── agents/
│       ├── checkout-security-auditor.md   ← 唯讀安全審查
│       └── e2e-test-writer.md             ← 可讀寫測試撰寫
├── src/
├── tests/
├── CLAUDE.md
└── package.json
```

### Phase 1：使用者啟動任務

```
使用者：用 agent teams 幫我實作購物車結帳功能，包含 Stripe 付款整合。
       前後端都要做，完成後要安全審查和 e2e 測試。
```

### Phase 2：建立團隊與任務

#### Team Lead 的決策

```
分析任務需求
    │
    ├── 需要搜尋現有程式碼模式 → Explore（內建）
    ├── 需要修改後端檔案 → general-purpose（內建）
    ├── 需要修改前端檔案 → general-purpose（內建）
    ├── 需要付款安全審查 → checkout-security-auditor（自訂）
    └── 需要撰寫 E2E 測試 → e2e-test-writer（自訂）
```

#### 建立團隊

```json
TeamCreate({
  "team_name": "checkout-feature",
  "description": "Implementing shopping cart checkout with Stripe payment"
})
```

#### 生成隊友

| 名稱 | agentType | 角色 |
|------|-----------|------|
| researcher | Explore | 研究現有程式碼 |
| backend-dev | general-purpose | 後端 API 開發 |
| frontend-dev | general-purpose | 前端 UI 開發 |
| sec-auditor | checkout-security-auditor | 付款安全審查 |
| test-writer | e2e-test-writer | E2E 測試撰寫 |

#### 建立共享任務清單

```
#1 [pending] 研究現有購物車與付款程式碼模式
   owner: null | blockedBy: []

#2 [pending] 設計 Stripe 整合的 API 端點
   owner: null | blockedBy: [#1]

#3 [pending] 實作後端結帳 API（/api/checkout）
   owner: null | blockedBy: [#2]

#4 [pending] 實作後端 Webhook 處理（Stripe events）
   owner: null | blockedBy: [#2]

#5 [pending] 實作前端結帳頁面 UI
   owner: null | blockedBy: [#1]

#6 [pending] 整合前端與後端結帳流程
   owner: null | blockedBy: [#3, #5]

#7 [pending] 安全審查所有付款相關程式碼
   owner: null | blockedBy: [#3, #4, #6]

#8 [pending] 撰寫 E2E 測試
   owner: null | blockedBy: [#6]
```

任務依賴圖：

```
#1 研究程式碼
├── #2 設計 API
│   ├── #3 後端結帳 API ──────┐
│   └── #4 後端 Webhook ──────┤
└── #5 前端結帳 UI ───────────┤
                              │
                    #6 前後端整合
                    ├── #7 安全審查
                    └── #8 E2E 測試
```

### Phase 3：團隊協作執行

完整時間線：

```
時間 ──────────────────────────────────────────────────────►

researcher (Explore，唯讀)
├── [認領 #1] 研究現有程式碼
│   搜尋 src/cart/、src/payment/、src/api/
│   分析現有購物車資料模型
├── [完成 #1]
│   SendMessage(broadcast):
│     "Found cart model at src/models/cart.ts,
│      existing payment utils at src/utils/payment.ts,
│      API pattern follows src/api/[resource]/route.ts"
├── [認領 #2] 設計 API 端點
│   設計 POST /api/checkout, POST /api/webhook/stripe
├── [完成 #2]
└── [idle] → 等待新任務或 shutdown

backend-dev (general-purpose)
├── [等待 #1, #2 完成]
├── [認領 #3] 實作 /api/checkout
│   建立 src/api/checkout/route.ts
│   實作 Stripe PaymentIntent、庫存檢查、價格驗證
├── [完成 #3]
├── [認領 #4] 實作 Webhook
│   建立 src/api/webhook/stripe/route.ts
│   實作簽名驗證、事件處理
├── [完成 #4]
│   SendMessage(message → sec-auditor):
│     "Backend checkout and webhook code is ready for review"
└── [idle]

frontend-dev (general-purpose)
├── [認領 #5]（#1 完成後可開始，與 backend-dev 並行）
│   建立 src/components/Checkout/
│   實作購物車摘要、Stripe Elements 表單
├── [完成 #5]
├── [等待 #3 完成]
├── [認領 #6] 整合前後端
│   串接 API、處理載入/錯誤狀態
├── [完成 #6]
│   SendMessage(message → sec-auditor):
│     "Frontend checkout integration complete"
│   SendMessage(message → test-writer):
│     "Full checkout flow is ready for e2e testing"
└── [idle]

sec-auditor (checkout-security-auditor，自訂，唯讀)
├── [等待 #3, #4, #6 完成]
├── [認領 #7] 安全審查
│   讀取所有付款相關程式碼
│   檢查 PCI-DSS 合規
│   檢查注入攻擊、race conditions
├── [完成 #7]
│   SendMessage(message → team-lead):
│     "Security review complete:
│      CRITICAL: No CSRF token on /api/checkout
│      WARNING: Missing rate limiting on payment endpoint
│      INFO: Consider adding idempotency key"
└── [idle]

test-writer (e2e-test-writer，自訂，可讀寫)
├── [等待 #6 完成]
├── [認領 #8] 撰寫 E2E 測試
│   讀取已實作的程式碼
│   建立 tests/e2e/checkout.spec.ts
│   撰寫 happy path、error、edge case 測試
│   執行測試、修復失敗
├── [完成 #8]
│   SendMessage(message → team-lead):
│     "E2E tests complete: 12 tests, all passing"
└── [idle]
```

並行執行圖（顯示時間重疊）：

```
          T1      T2      T3      T4      T5      T6      T7
          │       │       │       │       │       │       │
researcher ████████████
           #1 研究  #2 設計

backend-dev         ████████████████
                     #3 結帳API  #4 Webhook

frontend-dev  ██████████████████████████
              #5 前端UI          #6 整合

sec-auditor                              ████████
                                          #7 審查

test-writer                              ████████████
                                          #8 E2E 測試
```

### Phase 4：處理安全審查結果

Team Lead 收到 sec-auditor 的報告後動態追加任務：

```
sec-auditor 安全報告
    │
    ├── CRITICAL: 缺少 CSRF token
    │   → TaskCreate #9: "Add CSRF protection to /api/checkout"
    │     owner: null | blockedBy: []
    │
    ├── WARNING: 缺少 rate limiting
    │   → TaskCreate #10: "Add rate limiting to payment endpoints"
    │     owner: null | blockedBy: []
    │
    └── backend-dev（已 idle，被喚醒）
        → 自動認領 #9（最低 ID 優先）
        → 修復 CSRF 問題
        → 完成 #9，認領 #10
        → 修復 rate limiting
        → 完成 #10
        → SendMessage(message → sec-auditor):
            "Security fixes applied, please re-review"
```

### Phase 5：優雅關閉

```
所有任務完成
    │
    ▼
Team Lead 發送 shutdown_request 給所有隊友
    │
    ├── researcher:   approve（已 idle）
    ├── backend-dev:  approve
    ├── frontend-dev: approve
    ├── sec-auditor:  approve
    └── test-writer:  approve
    │
    ▼
TeamDelete("checkout-feature")
    → 清除 ~/.claude/teams/checkout-feature/
    → 清除 ~/.claude/tasks/checkout-feature/
    │
    ▼
Team Lead 向使用者回報最終結果
```

---

## 自訂代理設計要點

| 要點 | 說明 |
|------|------|
| **唯讀 vs 可讀寫** | 在 systemPrompt 中明確定義。安全審查類代理應設為唯讀，開發/測試類代理需要可讀寫 |
| **依賴管理** | 用 `blockedBy` 確保自訂代理在前置工作完成後才開始（如安全審查等所有程式碼完成） |
| **訊息驅動** | 上游隊友完成後主動通知下游自訂代理，減少等待 |
| **動態追加任務** | 自訂代理的發現（如安全問題）可觸發新任務，由其他隊友處理 |
| **CLAUDE.md 對齊** | Agent Creation Architect 會讀取 CLAUDE.md，確保自訂代理符合專案規範 |
| **團隊共享** | `.claude/agents/` 提交到 Git 後全團隊共用相同的代理定義 |
| **自治認領** | 自訂代理與內建代理一樣，自主從 TaskList 認領可用任務（最低 ID 優先） |

---

## 工具集對照表

| 隊友 | 代理類型 | Bash | Read | Write | Edit | Glob | Grep | SendMessage |
|------|---------|------|------|-------|------|------|------|-------------|
| researcher | Explore（內建） | RO | ✓ | ✗ | ✗ | ✓ | ✓ | ✓ |
| backend-dev | general-purpose（內建） | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| frontend-dev | general-purpose（內建） | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| sec-auditor | checkout-security-auditor（自訂） | RO | ✓ | ✗ | ✗ | ✓ | ✓ | ✓ |
| test-writer | e2e-test-writer（自訂） | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

`RO` = Read-Only（唯讀 bash，禁止修改命令）

**注意**：自訂代理的工具集由其 `systemPrompt` 中的描述決定。如果 systemPrompt 聲明「read-only access」，Claude Code 會將其視為唯讀代理，限制其工具存取。

---

## 來源檔案索引

| 檔案 | Token 數 | 相關內容 |
|------|---------|---------|
| `agent-prompt-agent-creation-architect.md` | 1110 | 自訂代理建立流程 |
| `tool-description-teammatetool.md` | 1642 | 團隊建立、代理類型選擇 |
| `tool-description-sendmessagetool.md` | 1241 | 隊友通訊機制 |
| `tool-description-taskcreate.md` | 558 | 任務建立與依賴管理 |
| `tool-description-tasklist-teammate-workflow.md` | 133 | 隊友任務認領流程 |
| `system-prompt-teammate-communication.md` | 127 | 隊友通訊規則 |
| `system-reminder-team-coordination.md` | 247 | 團隊協調提醒 |
| `system-reminder-team-shutdown.md` | 136 | 團隊關閉提醒 |
