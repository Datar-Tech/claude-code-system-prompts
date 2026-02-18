# Agent Teams 通訊與生命週期管理

> 基於 Claude Code v2.1.45 系統提示的深度分析，詳解 Agent Teams 的訊息系統、狀態管理與生命週期。

## 目錄

- [通訊規則總覽](#通訊規則總覽)
- [訊息類型詳解](#訊息類型詳解)
  - [message（點對點）](#message點對點)
  - [broadcast（全體廣播）](#broadcast全體廣播)
  - [shutdown 協議](#shutdown-協議)
  - [plan approval 協議](#plan-approval-協議)
- [通訊成本與策略](#通訊成本與策略)
- [Idle 狀態完整解析](#idle-狀態完整解析)
- [生命週期管理](#生命週期管理)
  - [啟動階段](#啟動階段)
  - [運行階段](#運行階段)
  - [關閉階段](#關閉階段)
- [通訊嚴格規則](#通訊嚴格規則)
- [相關系統提示來源](#相關系統提示來源)

---

## 通訊規則總覽

Agent Teams 的通訊系統基於一個核心原則：**隊友的純文字輸出對團隊不可見**。所有跨代理通訊必須透過 `SendMessage` 工具。

來自 `system-prompt-teammate-communication.md`：

> *「IMPORTANT: You are running as an agent in a team. To communicate with anyone on your team: Use the SendMessage tool. Just writing a response in text is not visible to others on your team - you MUST use the SendMessage tool.」*

---

## 訊息類型詳解

### message（點對點）

最常用的訊息類型，一對一通訊。

```json
{
  "type": "message",
  "recipient": "researcher",
  "content": "Found the auth module at src/auth/. Please investigate the token refresh logic.",
  "summary": "Auth module location for investigation"
}
```

| 欄位 | 必填 | 說明 |
|------|------|------|
| `type` | 是 | 固定為 `"message"` |
| `recipient` | 是 | 隊友名稱（不是 UUID） |
| `content` | 是 | 訊息正文 |
| `summary` | 是 | 5-10 字摘要，在 UI 中顯示為預覽 |

**遞送機制：**
- 收件人在 idle → 直接遞送，喚醒收件人
- 收件人在 mid-turn → 排入佇列，turn 結束後自動遞送
- UI 顯示通知：「[sender name] 的訊息等待中」

### broadcast（全體廣播）

向團隊所有成員同時發送同一訊息。

```json
{
  "type": "broadcast",
  "content": "Critical: blocking bug found in shared utils, stop all work",
  "summary": "Critical blocking issue found"
}
```

| 欄位 | 必填 | 說明 |
|------|------|------|
| `type` | 是 | 固定為 `"broadcast"` |
| `content` | 是 | 訊息正文 |
| `summary` | 是 | 5-10 字摘要 |

**成本警告**（來自 `tool-description-sendmessagetool.md`）：

> *「WARNING: Broadcasting is expensive. Each broadcast sends a separate message to every teammate, which means: N teammates = N separate message deliveries. Each delivery consumes API resources. Costs scale linearly with team size.」*

**合法用途：**
- 需要全體立即關注的緊急問題（如「發現阻塞性 bug，全體停工」）
- 影響每個隊友的重大公告

**不應使用 broadcast 的情境：**
- 回覆特定隊友
- 一般的工作進度更新
- 只和部分隊友相關的發現
- 任何不需要全體關注的訊息

### shutdown 協議

兩階段的優雅關閉協議。

**Step 1 — 發送關閉請求：**

```json
{
  "type": "shutdown_request",
  "recipient": "researcher",
  "content": "All tasks complete, wrapping up the session"
}
```

**Step 2a — 隊友批准：**

```json
{
  "type": "shutdown_response",
  "request_id": "abc-123",
  "approve": true
}
```

批准後：系統發送確認給 leader，隊友的進程終止。

**Step 2b — 隊友拒絕：**

```json
{
  "type": "shutdown_response",
  "request_id": "abc-123",
  "approve": false,
  "content": "Still working on task #3, need 5 more minutes"
}
```

拒絕後：leader 收到拒絕原因，等待隊友完成後重新請求。

**關鍵細節**（來自 `tool-description-sendmessagetool.md`）：

> *「When you receive a shutdown request as a JSON message with `type: "shutdown_request"`, you **MUST** respond to approve or reject it. Do NOT just acknowledge the request in text - you must actually call this tool.」*

### plan approval 協議

當隊友設有 `plan_mode_required` 時，他們的實作計畫需要 leader 審批。

**觸發：** 隊友呼叫 `ExitPlanMode` → 系統自動發送 `plan_approval_request` 給 leader。

**Leader 批准：**

```json
{
  "type": "plan_approval_response",
  "request_id": "abc-123",
  "recipient": "researcher",
  "approve": true
}
```

批准後：隊友自動退出 Plan Mode，開始實作。

**Leader 拒絕（附帶反饋）：**

```json
{
  "type": "plan_approval_response",
  "request_id": "abc-123",
  "recipient": "researcher",
  "approve": false,
  "content": "Please add error handling for the API calls"
}
```

拒絕後：隊友收到反饋，修改計畫，可再次提交。

---

## 通訊成本與策略

### 成本排序

```
低成本 ◄──────────────────────────────► 高成本

TaskUpdate    message     broadcast
(狀態更新)   (一對一)      (N 倍成本)
系統自動處理   可控成本     慎用
```

### 場景對照表

| 情境 | 錯誤做法 | 正確做法 |
|------|---------|---------|
| 任務完成 | broadcast "Task 3 done!" | `TaskUpdate(completed)` + message 給 lead |
| 發現 bug | 文字輸出 "I found a bug..." | message 給相關隊友 |
| 需要另一個代理的產出 | 持續 polling TaskList | message 請求 + 等待回覆 |
| 全體停工 | 逐一 message 每個人 | broadcast（此時是正當用途） |
| 任務狀態更新 | 發訊息說「我開始做了」 | `TaskUpdate(status: "in_progress")` |

### 最小化通訊的原則

1. **任務狀態優先於訊息** — 完成任務用 `TaskUpdate(completed)`，不用 SendMessage 通知
2. **點對點為主** — 大多數溝通只涉及兩方
3. **broadcast 僅限緊急情況** — 如發現阻塞性 bug
4. **不發送結構化 JSON 狀態訊息** — 系統自動處理 idle/completed 通知

---

## Idle 狀態完整解析

系統提示中 idle 狀態的說明篇幅大，因為這是最容易被誤解的概念。

### 基本事實

| 事實 | 說明 |
|------|------|
| idle 何時觸發 | 隊友**每次 turn 結束後**自動進入 |
| idle ≠ 完成 | 只是「等待輸入」 |
| idle ≠ 離線 | idle 的隊友**可以接收訊息**，收到後會被喚醒 |
| idle 通知是自動的 | 系統自動發送，不需要回應 |

來自 `tool-description-teammatetool.md`：

> *「Teammates go idle after every turn — this is completely normal and expected. A teammate going idle immediately after sending you a message does NOT mean they are done or unavailable. Idle simply means they are waiting for input.」*

> *「Be patient with idle teammates! Don't comment on their idleness until it actually impacts your work.」*

### Idle 狀態流程圖

```
隊友完成一個 turn
        │
        ▼
  系統自動設為 idle
        │
        ├── 發送 idle 通知給 team lead
        │   （包含 peer DM 摘要，如果有的話）
        │
        └── 等待輸入
              │
         ┌────┴────┐
         │         │
    收到訊息    收到任務指派
    (message)   (TaskUpdate)
         │         │
         ▼         ▼
      被喚醒     被喚醒
      處理訊息   開始任務
         │         │
         ▼         ▼
      turn 結束  turn 結束
         │         │
         ▼         ▼
      再次 idle  再次 idle
```

### Peer DM 可見性

當隊友 A 發 DM 給隊友 B 時，team lead 會在 A 的 idle 通知中看到一個簡短摘要。

> *「When a teammate sends a DM to another teammate, a brief summary is included in their idle notification. This gives you visibility into peer collaboration without the full message content. You do not need to respond to these summaries — they are informational.」*

---

## 生命週期管理

### 啟動階段

```
1. TeamCreate { team_name: "my-project" }
   → 建立 ~/.claude/teams/my-project/config.json
   → 建立 ~/.claude/tasks/my-project/

2. 建立任務（含依賴關係）
   TaskCreate { subject: "...", blockedBy: [...] }
   TaskCreate { subject: "...", blockedBy: [...] }

3. 啟動隊友
   Task { name: "researcher", subagent_type: "Explore",
          team_name: "my-project", prompt: "..." }
   Task { name: "coder", subagent_type: "general-purpose",
          team_name: "my-project", prompt: "..." }
```

**建議：**
- 先建立所有任務和依賴，再啟動隊友
- 研究類隊友先啟動（產出不依賴其他人）
- 實作類隊友可同時啟動（會自動認領無阻塞的任務）

### 運行階段

隊友的自然循環：

```
認領任務 → TaskUpdate(in_progress) → 工作 → TaskUpdate(completed)
    │
    ▼
檢查 TaskList → 有可用任務？
    ├── 是 → 認領下一個（優先低 ID）
    └── 否 → idle（等待新任務或訊息喚醒）
```

**任務認領規則**（來自 `tool-description-tasklist-teammate-workflow.md`）：

1. 完成當前任務後呼叫 `TaskList`
2. 尋找：`status: "pending"` + 無 owner + `blockedBy` 為空
3. **優先 ID 最小的任務**（低 ID 通常為後續任務建立上下文）
4. 用 `TaskUpdate(owner: "my-name")` 認領
5. 全部被阻塞 → 通知 team lead 或幫助解除阻塞

### 關閉階段

完整的關閉流程（來自 `system-reminder-team-shutdown.md`）：

```
1. 確認所有任務完成

2. 逐一發送 shutdown_request
   SendMessage { type: "shutdown_request", recipient: "researcher" }
   SendMessage { type: "shutdown_request", recipient: "coder" }

3. 等待每個 shutdown_response
   ├── approve: true → 隊友進程終止
   └── approve: false → 等待完成 → 再次請求

4. 所有隊友關閉後
   TeamDelete → 清理團隊和任務目錄

5. 向使用者報告最終結果
```

**重要限制：**

- `TeamDelete` 在有活躍成員時**會失敗**
- 在非互動模式下，使用者**無法收到回覆**直到團隊完全關閉

來自 `system-reminder-team-shutdown.md`：

> *「You are running in non-interactive mode and cannot return a response to the user until your team is shut down. You MUST shut down your team before preparing your final response.」*

---

## 通訊嚴格規則

以下規則直接來自系統提示，違反會導致通訊失敗或行為異常：

| 規則 | 來源 | 後果 |
|------|------|------|
| 純文字輸出對團隊**不可見** | `system-prompt-teammate-communication.md` | 隊友永遠收不到 |
| **永遠用名稱稱呼隊友**，不用 UUID | `tool-description-teammatetool.md` | 訊息無法送達 |
| **不要發送結構化 JSON 狀態訊息** | `tool-description-teammatetool.md` | 造成解析混亂 |
| 用 `TaskUpdate` 標記完成，不用訊息 | `tool-description-teammatetool.md` | 依賴任務不會解除阻塞 |
| shutdown_request 必須用 tool 回應 | `tool-description-sendmessagetool.md` | 純文字確認無效 |
| 不用終端工具查看團隊活動 | `tool-description-teammatetool.md` | 用 SendMessage 直接溝通 |
| 回報隊友訊息時不需引用原文 | `tool-description-teammatetool.md` | 原文已在 UI 中呈現 |

---

## 相關系統提示來源

本文分析基於以下系統提示檔案（均位於 `system-prompts/` 目錄）：

| 檔案 | Token 數 | 內容 |
|------|---------|------|
| `tool-description-teammatetool.md` | ~1642 | TeamCreate 工具描述、團隊工作流、idle 狀態說明 |
| `tool-description-sendmessagetool.md` | ~800+ | SendMessage 工具描述、四種訊息類型 |
| `tool-description-teamdelete.md` | ~100 | TeamDelete 工具描述 |
| `tool-description-tasklist-teammate-workflow.md` | ~133 | 隊友任務認領工作流 |
| `tool-description-taskcreate.md` | ~400+ | TaskCreate 工具描述、任務欄位定義 |
| `system-prompt-teammate-communication.md` | ~127 | 隊友通訊基本規則 |
| `system-reminder-team-coordination.md` | ~200+ | 團隊協調提醒（注入隊友身份和路徑） |
| `system-reminder-team-shutdown.md` | ~100 | 團隊關閉流程提醒 |
| `system-reminder-delegate-mode-prompt.md` | ~200+ | Delegate Mode 工具限制 |
| `system-reminder-exited-delegate-mode.md` | ~50 | 退出 Delegate Mode 通知 |
