# 匯入流程怎麼把 Claude 對話寫成 thread

Ticket：https://github.com/AI-Reletive-Project/t3code/issues/2
程式碼：`本機 clone`（main，commit 629d5083b），以下路徑都相對於這個 clone。只讀原始碼，沒有執行。

## 結論

1. **怎麼轉**：`AgentSessionScanner.recentThreads` 讀出專案底下近 30 天的紀錄檔，每個檔案得到一個結果：`Importable`、`AlreadyImported`、`Duplicate` 或 `Skipped`（`apps/server/src/project/AgentSessionScanner.ts:221-231`、`:2131-2253`）。Claude 紀錄只取 `type` 為 `user`/`assistant`、而且不是 sidechain/meta/compact summary 的紀錄，也只取純文字區塊（tool_use、tool_result、thinking 都丟掉）（`AgentSessionScanner.ts:308-322`、`:342-344`、`:594-605`）。`AgentSessionImporter.importRecentAgentThreads` 再把 `Importable` 寫成 thread（`apps/server/src/project/AgentSessionImporter.ts:270-496`）。
2. **產生哪些事件**：新 thread 在**同一次** `EventSinkV2.write` 裡寫入這些事件：`thread.created`（`historyOrigin: "v1_import"`、`settledOverride: "settled"`），有省略的訊息時加一個 `system_notice` 的 `turn-item.updated`，每則訊息各一個 `message.updated` 加一個 `turn-item.updated`（`user_message`/`assistant_message`，`runId: null`），最後是 `provider-thread.updated`（`AgentSessionImporter.ts:423-475`、`:112-195`）。
3. **存在哪裡**：事件寫進 SQLite 的 `orchestration_events`（`event_id` 有 UNIQUE 限制），同一個交易裡投影到 `orchestration_v2_projection_messages`、`orchestration_v2_projection_turn_items` 等表，commit 後透過 PubSub 推給訂閱者（`apps/server/src/orchestration-v2/EventSink.ts:467-521`、`:303-312`；`apps/server/src/persistence/OrchestrationEventStore.ts:475-511`；`apps/server/src/persistence/Migrations/001_OrchestrationEvents.ts:10`；`apps/server/src/orchestration-v2/ProjectionStore.ts:2864-2973`）。resume 資訊與「已匯入」紀錄存在 `provider_session_runtime` 表（`AgentSessionImporter.ts:306-322`）。
4. **怎麼記錄「已匯入」**：`ProviderSessionRuntimeRepository.recordImportedTranscript` 把 transcript identity（`filePath`、`size`、`mtimeMs`、`device`、`inode`、`birthtimeMs`，外加 provider、instance、sessionId）寫進該 thread 的 `runtime_payload_json.importedTranscripts` 陣列；同一個 instance 加同一個 filePath 只保留最新一筆（`apps/server/src/persistence/ProviderSessionRuntime.ts:238-283`；`packages/contracts/src/agentSessions.ts:8-20`）。下次匯入時，六個欄位全部相同才算 `AlreadyImported`（`AgentSessionScanner.ts:821-837`、`:2144-2157`）。`v1_import` 是 thread 上的 `historyOrigin` 值，代表「歷史是匯入的、不屬於任何 run」，不是匯入紀錄本身（`packages/contracts/src/orchestrationV2.ts:102`）。
5. **重新匯入的行為（跟 ticket 背景描述的出入）**：這個 commit 對**已存在**的 `v1_import` thread 不會寫新事件。比對結果分兩種：(a) 已存的訊息跟新解析出的訊息逐則相同，就只補記 identity；(b) 有任何不同（例如紀錄檔多了新訊息），就直接 `return false`，算成 skipped，**連 identity 都不記**（`AgentSessionImporter.ts:337-353`）。測試 `does not record a changed transcript as imported without its new messages` 鎖住了這個行為（`AgentSessionImporter.test.ts:579-618`）。所以背景說的「只記 identity、不寫新事件」只適用於內容沒變的情況；內容變了的情況是兩件事都不做。
6. **有沒有現成的「追加訊息」入口**：**沒有可以直接用的公開入口。**
   - WS 只開放 `agentSessionsScan`、`agentSessionsImport`（`apps/server/src/ws.ts:3024-3026`），而 import 對內容有變的 thread 會跳過（見第 5 點）。
   - `importNativeHistory` 會把訊息寫進既有 thread，但條件是 `historyOrigin !== "v1_import"` 而且還沒跑過 run（`AgentSessionImporter.ts:97-102`、`:507-597`），所以碰不到已匯入的 thread。它的 message ID 也從 index 0 開始編，用在追加上會跟既有 ID 衝突。
   - 可以重用的最底層積木：`EventSinkV2.write`（`EventSink.ts:100-109`）、importer 內部未 export 的 `messageEvents` 產生器（`AgentSessionImporter.ts:112-195`），以及 `ThreadCommandExecutor.withLock` 這種「鎖住 thread、重讀、寫入」的寫法（`AgentSessionImporter.ts:521-546`）。推論：同步功能要自己寫一個 service 來組合這些積木。
   - **刪除最舊訊息沒有任何現成機制**：V2 domain event 沒有 message 或 turn-item 的刪除型別，只有 `*.updated`（`packages/contracts/src/orchestrationV2.ts:1982-2125`）；投影對 messages、turn_items 也只做 upsert（`ProjectionStore.ts:2864-2905`、`:2939-2973`）。正式程式碼裡唯一的 `DELETE FROM orchestration_v2_projection_messages` 在測試工具裡（`apps/server/src/orchestration-v2/testkit/ProjectionReplay.ts:110-112`）。推論：「最多 200 則、移除最舊」需要新增事件型別或另一種投影刪除機制。
7. **200 則上限在哪裡套用**：只在 scanner 解析階段，常數 `MAX_IMPORTED_MESSAGES = 200`（`AgentSessionScanner.ts:87`），分兩層套用：
   - 讀檔時，`createClaudeHistoryWindow` 只保留最近 200 筆紀錄，另外固定留住第一則 user 紀錄，其餘計入 `droppedMessageCount`（`AgentSessionScanner.ts:723-746`、`:1274-1276`）；
   - 解析時，`retainMessage` 超過 200 就 `shift()` 掉最舊的（`:549-561`）。最後如果第一則 user 訊息被擠掉，就變成「第一則 user + 最後 199 則」（`:670-675`），省略的數量放在 `omittedMessageCount`（`:686`），importer 用它產生 system notice（`AgentSessionImporter.ts:434-461`）。
   - Importer、EventSink、投影都**沒有**上限；thread 寫入後也沒有任何地方再修剪。

## 細節

### 1. 進入點

- WS 方法 `agentSessionsImport` 呼叫 `agentSessionImporter.importRecentAgentThreads(input)`，輸入是 `{ projectId, expectedWorkspaceRoot? }`（`apps/server/src/ws.ts:3025-3026`；`packages/contracts/src/agentSessions.ts:51-55`）。
- `importNativeHistory` 只由 ACP 相關流程呼叫（`apps/server/src/provider/AcpRegistrySessions.ts:188`、`:256`），跟 Claude 紀錄檔匯入無關。

### 2. Importer 主流程（`AgentSessionImporter.ts:208-499`）

1. 找出專案，並確認 workspaceRoot 沒有變（`:211-227`）。
2. runtimeMode 預設用 full access；有 policy 上限時，改用專案預設值並壓在上限以下（`:228-242`）。
3. 從 `provider_session_runtime` 的所有列取出 `importedTranscripts`，只留屬於本專案的 thread，當作 `completedSources` 傳給 scanner（`:243-265`）。
4. 逐筆處理 scanner 的結果（`:270-496`）：
   - `Skipped`：計數加一。
   - `AlreadyImported`：只記下 threadId，不寫任何東西（`:280-283`）。
   - `Duplicate`：同一個 session 的另一個檔案。如果本次已經匯入這個 thread，就補記一筆 identity（`:284-289`）。
   - `Importable`：
     - Claude 的 sessionId 必須是 UUID，否則丟 `AgentSessionUnresumableSessionError`（`:293-301`）。
     - threadId 固定是 `import:${providerInstanceId}:${providerSessionId}`（`:277-279`），所以同一個 Claude session 永遠對應同一個 thread。
     - 準備 `provider_session_runtime` 列，`onConflict: "ignore"`，`resumeCursor = { threadId, resume: <claude sessionId> }`（`:306-322`）。
     - thread 已存在時：project 不同就報錯；`historyOrigin` 不是 `v1_import`（使用者已經在上面互動過）就報 `AgentSessionThreadModifiedError`；然後比對訊息（見結論第 5 點）（`:323-354`）。
     - thread 不存在時：組出 `appThread` 和 `providerThread`，一次寫入所有事件。`unlessNativeThreadOwned` 讓寫入在「另一個 thread 已經擁有這個 native session」時寫零筆（`:356-479`；`EventSink.ts:480-493`）。寫入成功後才 upsert runtime、記 identity（`:476-479`）。
   - 任何錯誤都只記 warning，算成 skipped（`:480-488`）。

### 3. 事件與 ID 格式（`AgentSessionImporter.ts:50`、`:112-195`、`:423-475`）

前綴 `IMPORT_EVENT_PREFIX = "agent-session-import:v2"`。`suffix` 是訊息在「匯入視窗」裡的 index，補零到 6 位。

| 事件 | event id | payload id |
|---|---|---|
| `thread.created` | `agent-session-import:v2:thread:<threadId>:created` | threadId |
| `turn-item.updated`（system_notice，選用） | `…:history-truncated:<threadId>` | 同左 |
| `message.updated` | `…:message:<threadId>:<suffix>` | MessageId `<threadId>:<suffix>` |
| `turn-item.updated`（user/assistant_message） | `…:turn-item:<threadId>:<suffix>` | 同左 |
| `provider-thread.updated` | `…:provider-thread:<providerThreadId>` | providerThreadId |

要注意的地方：
- user 訊息的 turn item 用 `inputIntent: "turn_start"`，`createdBy: "user"`；assistant 訊息的 message `createdBy: "agent"`（`:152-184`）。
- `ordinal` 欄位會被 `TurnItemPositionStore.normalize` 重算：`runId` 為 null 的 item 落在 0–999,999 區間，依寫入順序取 `MAX+1`（`apps/server/src/orchestration-v2/TurnItemPositionStore.ts:72-112`、`:143-151`；`EventSink.ts:389-410`）。推論：之後追加的 run-less item 仍然排在同一區間的尾端；但如果使用者已經在 T3 裡跑過 run（run 的 ordinal × 1,000,000 起算），追加的 run-less item 會排在那些 run 前面。
- **`event_id` 有 UNIQUE 限制**（`Migrations/001_OrchestrationEvents.ts:10`），寫入時也沒有 `ON CONFLICT`（`OrchestrationEventStore.ts:475-511`）。推論：同步功能如果沿用 `…:message:<threadId>:<suffix>` 這種依 index 產生的 ID，追加或改寫同一則訊息時會撞 UNIQUE，必須換成新的、穩定且唯一的 event id。MessageId 本身在投影表是 `ON CONFLICT(message_id) DO UPDATE`，可以重複 upsert（`ProjectionStore.ts:2893-2903`）。
- message ID 依 index 產生，而 index 是「200 則視窗內」的位置，不是紀錄檔裡的絕對位置（`AgentSessionImporter.ts:117-119`、`:462-463`）。推論：視窗往前滑動之後，同一個 index 會對到不同的訊息，不能直接拿來做增量同步的鍵。scanner 的 schema 也沒有解析 Claude 紀錄的 `uuid` 欄位（`AgentSessionScanner.ts:123-152`），要用它當穩定鍵就得擴充 schema。

### 4. 「已匯入」的記錄與判斷

- identity 由 `fs.stat` 組成：`filePath`、`size`、`mtimeMs`、`device`、`inode`（字串）、`birthtimeMs`（`AgentSessionScanner.ts:800-812`）。
- 比對時六個欄位全部相同才算同一份（`:821-837`）。所以紀錄檔一被追加內容（size 或 mtime 改變），下次匯入就不會是 `AlreadyImported`，會重新解析成 `Importable`，再走到 importer 的「thread 已存在」分支（`:2144-2167`）。
- 寫入的 SQL 會移除同一個 `providerInstanceId + filePath` 的舊項目，再附加新項目（`ProviderSessionRuntime.ts:238-283`）。一般的 `upsert` 不能改 `importedTranscripts`，只會保留舊值（`ProviderSessionRuntime.ts:141`、`:166-200`）。
- `scan` 也用這份紀錄：identity 沒變的已匯入檔案沿用匯入時的 cwd，不必重讀（`AgentSessionScanner.ts:1667-1679`）。
- 推論：同步功能每次追加之後，應該用新的 identity 呼叫 `recordImportedTranscript`。這樣使用者之後手動按匯入，會走 `AlreadyImported`，不會因為 identity 對不上又被當成內容有變的 thread 而跳過。

### 5. `historyOrigin: "v1_import"` 的影響

型別只有 `"native" | "v1_import"`（`packages/contracts/src/orchestrationV2.ts:102`）。`v1_import` 會影響這些地方：
- 是否還需要讀 native history（`AgentSessionImporter.ts:97-102`）。
- fork 時，run-less item 也算在 fork 點之前，會被複製（`ProjectionStore.ts:1510-1518`、`:2021`）。
- 在 T3 裡第一次對它發訊息時，會把 run-less 匯入歷史做成 context handoff 交給 provider（`apps/server/src/orchestration-v2/Orchestrator.ts:1346-1358`、`:2783-2786`）。
- 匯入產生的 `thread.created` 不會透過 AgentAwarenessRelay 發出（`apps/server/src/relay/AgentAwarenessRelay.ts:83-91`）。
- 一旦 thread 不再是 `v1_import`，重新匯入就會被 `AgentSessionThreadModifiedError` 擋下（`AgentSessionImporter.ts:334-336`）。

### 6. Scanner 的其他限制（跟即時同步有關）

- 只看 mtime 在 30 天內的檔案（`AgentSessionScanner.ts:84`、`:2051-2056`）。每次匯入最多讀 100 個新檔案、總共 4 GiB（`:89-90`、`:2078-2079`、`:2158-2166`）。選出的歷史最多 32 MiB（`:88`、`:1247-1253`）。
- 讀檔前後都比對 identity，不一致就回傳 null，結果變成 `Skipped`（`:1235-1237`、`:1336-1344`、`:2167-2170`）。推論：正在寫入中的紀錄檔有機會因此被跳過，即時同步要能重試。
- 候選檔案清單會快取，只有在 `scan` 時或上次收集被截斷時才重新收集（`:1884-1887`、`:1996-1998`）。推論：伺服器啟動後新出現的 session 檔，要等使用者再按一次 scan 才找得到。
- 同一個 session 對應多個檔案時，第二個以後的檔案標成 `Duplicate`（`:2237-2240`）。

### 7. 寫入管線（EventSink）

`EventSinkV2.write`（`EventSink.ts:467-521`）在同一個 SQL 交易裡依序做這些事：
1. 檢查 native thread 的擁有者。
2. normalize turn item 的 ordinal。
3. `eventStore.append`。
4. `projectionStore.apply` 每個事件，並更新 projection metadata。

commit 之後 `publishStoredEvents` 把事件推到 live PubSub（`:303-312`）；`stream()` 的訂閱者（含 bounded RPC 訂閱）會即時收到（`:994-1038`）。推論：只要經由 EventSink 寫入，已開著該 thread 的客戶端（包括手機）就會即時看到新訊息，不需要另外推播。

### 8. 另一個同名但不同的匯入器

`apps/server/src/orchestration-v2/legacy/LegacyV1ThreadImporter.ts` 也寫 `historyOrigin: "v1_import"`（`:234`），但它的前綴是 `migration:v1`（`:38`），用途是把舊版 V1 的 thread 遷移到 V2，跟 Claude 紀錄檔無關。
