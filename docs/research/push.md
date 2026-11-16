# thread 的新訊息怎麼即時送到手機

Ticket：https://github.com/AI-Reletive-Project/t3code/issues/3
程式碼依據：本機 clone `本機 clone`（main，commit 629d5083b）。下文路徑都相對於這個 clone。
標「推論」的段落是從程式碼推出來的，沒有實際跑過。

## 結論

1. **路徑**：伺服器用 `EventSinkV2.write` 寫入事件，交易提交後把事件發到同一個行程裡的 PubSub（`apps/server/src/orchestration-v2/EventSink.ts:280`、`:288`、`:303-312`）。手機打開 thread 時，透過 WebSocket RPC `orchestration.subscribeThread` 訂閱這個 thread。伺服器先送一份 snapshot，或從 `afterSequence` 補送漏掉的事件，接著把 PubSub 的即時事件一筆一筆串流下去（`apps/server/src/ws.ts:897-1127`）。手機端用 `packages/client-runtime` 的 `applyOrchestrationV2ProjectionEvent` 把事件套到本機的 projection 上，畫面再從 `visibleTurnItems` 產生（`apps/mobile/src/state/use-thread-composer-state.ts:278-284`）。T3 Connect 只是另一條連線路線：relay 授權後給出端點，手機連的還是同一個 `/ws` RPC socket，資料格式完全相同（`packages/client-runtime/src/authorization/remote.ts:269-275`、`packages/client-runtime/src/connection/resolver.ts:180-196`）。
2. **伺服器端追加訊息，手機不改程式能不能即時顯示**：**可以**，但有三個條件：
   - (a) 必須在伺服器行程內用 `EventSinkV2.write` 寫入 `message.updated` 和 `turn-item.updated` 事件。直接改 SQLite 不會觸發 PubSub，手機收不到即時事件。
   - (b) 寫法比照現有匯入流程：`runId: null`，`ordinal` 比現有最大值還大，`id` 不能重複（`apps/server/src/project/AgentSessionImporter.ts:112-195`、`:523-546`）。
   - (c) 手機正在看這個 thread，也就是訂閱存在時才是即時。沒在看的話，下次打開時用 `afterSequence` 補送，一樣看得到。
3. **伺服器移除最舊的訊息，手機顯示會不會跟著更新**：**一般情況不會**。
   - 協定裡沒有「刪除單則 message 或 turn item」的事件，只有 `*.updated` 和 `*.created` 這類 upsert 事件（`packages/contracts/src/orchestrationV2.ts:1979-2128`）。
   - 手機端對 `message.updated` 和 `turn-item.updated` 只會做 upsert（`packages/client-runtime/src/state/orchestrationV2Projection.ts:415-454`）。
   - 手機會把 projection 存進本機快取，重開時用 `afterSequence` 續傳，伺服器只補送事件、不重送 snapshot（`packages/client-runtime/src/state/threads.ts:249-290`、`:1090-1107`；`apps/server/src/ws.ts:1025-1123`）。所以就算重開 app，已經刪掉的訊息也會留在手機上。
   - 只有在伺服器決定送新 snapshot 時，手機的整份 projection 才會被換掉（`threads.ts:655-676`、`:439-470`）。
   - 推論：手機預設只載入最近一個視窗（約 75 列，或最近 10 個 user turn；`apps/server/src/orchestration-v2/threadHistoryPaging.ts:16-21`、`:32`）。thread 上限 200 則時，被移除的最舊訊息通常本來就不在手機目前載入的範圍內，所以實際上看不出差別。使用者往上捲、載入更早的歷史之後，才會看到「已刪除卻還在」的訊息。

## 細節

### 1. 伺服器端：事件怎麼產生、怎麼發布

- 所有寫入都經過 `EventSinkV2`。它在初始化時建立一個行程內的 `PubSub.unbounded`（`EventSink.ts:280`）。`publishLiveEvents` 把提交完成的事件 `publishAll` 出去（`EventSink.ts:285-295`），`publishStoredEvents` 的順序是先 `eventStore.publishCommitted`，再 `publishLiveEvents`（`EventSink.ts:303-312`）。
- 提交和發布由一條 `Semaphore(1)` 串行處理，這樣客戶端才不會因為順序錯亂而丟掉較早的 sequence（`EventSink.ts:314-315`）。
- `EventSinkV2.stream({threadId, afterSequence, bounded})` 的流程是：先訂閱 PubSub，再從資料庫補送到 highWater，最後接上即時事件，並用 `threadId` 過濾（`EventSink.ts:994-1038`）。RPC 訂閱者用 `bounded: true`，緩衝區上限是 1000 筆或 8 MB，超過就失敗，錯誤訊息要客戶端「從最後收到的 sequence 續傳」（`LiveStreamBudget.ts:18-19`、`:67-84`）。
- `OrchestratorV2.streamStoredEventsFrom` 就是 `eventSink.stream({...input, bounded: true})`（`Orchestrator.ts:539-544`、`:16410-16418`）。
- **推論**：PubSub 只存在於伺服器行程的記憶體裡。另一個行程直接寫 SQLite 不會觸發即時推送，要等手機下次訂閱時才會被 `afterSequence` 的補送讀到（前提是那些列有正確的 sequence）。所以同步器應該放在伺服器行程內，透過 `EventSinkV2.write` 寫入。

### 2. WebSocket RPC：`orchestration.subscribeThread`

- 方法名稱定義在 `packages/contracts/src/orchestrationV2.ts:3840`，輸入和輸出的 schema 在 `:4558-4561`，RPC 定義在 `packages/contracts/src/rpc.ts:1960-1962`，伺服器的 handler 註冊在 `apps/server/src/ws.ts:2282-2287`。
- 輸入參數有 `threadId`、`afterSequence`、`requestCompletionMarker`、`acceptBoundedSnapshot` 等（`orchestrationV2.ts:4054-4064`）。
- 串流項目有四種：`synchronized`、`snapshot`、`event`，以及未知事件類型解碼後的 `unknown-event`（`orchestrationV2.ts:4165-4188`、`:4132-4163`）。
- 伺服器處理流程在 `ws.ts:897-1127`：
  - 沒有 `afterSequence` 時走 `snapshotThenLive`：先送 snapshot（bounded 視窗或完整內容），再送 `synchronized`，最後接上 `eventStreamFrom(snapshotSequence)`（`ws.ts:981-1022`、`:1126`）。
  - 有 `afterSequence` 時，預設只補送事件，再接上即時串流（`ws.ts:1091-1123`）。只有下列情況才改送 snapshot：
    - `afterSequence > highWater`，或 cursor 已經早於被清掉的範圍（`afterSequence < prunedThrough`）（`ws.ts:1047-1052`、`:1093-1108`）。
    - 補送超過 `THREAD_RESUME_MAX_REPLAY_EVENTS = 128` 筆，或超過大小上限（`ws.ts:1071-1076`；`ThreadStream.ts:18`）。
    - 補送範圍內有 create 事件（`ws.ts:1077-1090`）。
    - `decideThreadResume` 判定要用 snapshot（`ws.ts:1110-1118`）。
- 即時串流在 50 ms 的視窗內，會把「同一個串流中的 assistant 或 reasoning 項目」的多筆更新合併成一筆。已完成（非 streaming）的 `message.updated` 和 `turn-item.updated` 不會被合併（`ThreadLiveEventCoalescer.ts:18`、`:24-48`）。

### 3. T3 Connect 轉送

- 客戶端的 relay broker 拿到 DPoP 授權後，回傳的 `PreparedConnection` 和直連的格式一樣（`packages/client-runtime/src/connection/resolver.ts:180-196`）。WebSocket URL 由端點的 `wsBaseUrl` 算出來，路徑為空時補成 `/ws`（`packages/client-runtime/src/authorization/remote.ts:269-275`）。
- 伺服器端的 T3 Connect 是一條受管的 cloudflared tunnel，把公開主機名稱導到本機服務（`apps/server/src/cloud/managedTunnelStartup.ts:116`、`:237`）。
- **推論**：relay 和 tunnel 只負責轉送位元組，不會解讀 orchestration 事件。所以「新增」和「移除」在直連和 T3 Connect 下的行為完全一樣，本文的結論兩種路線都適用。

### 4. 手機端：怎麼套用事件、怎麼顯示

- 手機的 thread 狀態直接使用 client-runtime 的 `createEnvironmentThreadStateAtoms` 和 `createEnvironmentThreadDetailAtoms`（`apps/mobile/src/state/threads.ts:3-12`、`:41-44`）。畫面上的 feed 是 `buildThreadFeed(selectedThreadVisibleTurnItems, …)`（`apps/mobile/src/state/use-thread-composer-state.ts:278-284`；`apps/mobile/src/state/use-thread-detail.ts:67-86`）。也就是說，**顯示內容只看 `projection.visibleTurnItems`**。
- 訂閱邏輯在 `packages/client-runtime/src/state/threads.ts:999-1129`：
  - 本機有資料時，帶 `afterSequence` 續傳（`:1090-1107`）。
  - 冷啟動、本機沒資料時，先用 HTTP 抓 snapshot，或直接要 socket 的 bounded snapshot（`:1018-1088`）。
  - 前景喚醒或串流錯誤時會重新訂閱（`:957-961`、`:1112-1128`）。
- 套用事件：
  - `applyItems` 和 `applyEventsLocked` 會依 sequence 去重，未知事件類型只推進 cursor、不做其他處理（`threads.ts:523-545`、`:683-705`）。
  - 每一筆事件交給 `applyOrchestrationV2ProjectionEvent` 處理（`threads.ts:586`）。
  - `snapshot` 用 `setThread` **整份替換** projection（`threads.ts:655-676`、`:439-470`）。
- `applyOrchestrationV2ProjectionEvent`（`packages/client-runtime/src/state/orchestrationV2Projection.ts:286-467`）：
  - `message.updated` 用 `upsertEntity` 更新（`:415-416`、`:57-66`）。
  - `turn-item.updated` 先 upsert 到 `turnItems`，再依可見性 upsert 進 `visibleTurnItems`，或從中移除（`:419-454`）。新項目依 `ordinal` 插入正確位置（`:197-261`）。
  - 部分視窗（bounded）的過濾規則：`runId === null` 的新項目，如果 `ordinal` 不超過 `latestLocalTurnOrdinal`，或比目前最舊的本機列還小，會被當成「還沒載入的舊歷史」丟掉（`:148-195`）。所以追加的訊息 **ordinal 必須遞增**。收到更大的 ordinal 時，水位會跟著更新（`threads.ts:591-598`）。

### 5. 「追加」要怎麼寫才會被手機顯示（推論，依現有匯入流程）

- 現有的 native history 匯入就是「伺服器自己追加、沒有 run 的訊息」，可以直接當範本。每則訊息寫一個 `message.updated`（payload 是 `OrchestrationV2ConversationMessage`）和一個 `turn-item.updated`（`user_message` 或 `assistant_message`，`runId: null`，`ordinal = index + 1`，`status: "completed"`，`streaming: false`）（`AgentSessionImporter.ts:112-195`）。寫入方式是在 `threadCommands.withLock` 裡呼叫 `eventSink.write`（`AgentSessionImporter.ts:523-546`）。
- 因為這條路徑走 `EventSinkV2.write`，正在訂閱的手機會即時收到兩筆事件。turn item 依可見性規則是可見的（runId 為 null，不會受到 rolled_back 或 queued 的規則影響；`packages/shared/src/orchestrationV2Timeline.ts:41-73`），所以會出現在 feed 尾端。手機端不用改程式。
- 要注意的地方：
  - `importNativeHistory` 只在「歷史尚未匯入」時執行一次（`AgentSessionImporter.ts:507-520`、`:99-102`），不能拿來做增量同步。同步器必須自己算出下一個 ordinal 和不重複的 id。
  - 一次寫入太多事件時，手機續傳會超過 128 筆，伺服器改送 snapshot（`ws.ts:1071-1076`）。這不影響正確性。

### 6. 「移除最舊」為什麼不會反映到手機

- 事件聯集 `OrchestrationV2DomainEvent` 裡沒有 message 或 turn item 的刪除事件（`orchestrationV2.ts:1979-2128`）。手機 reducer 裡會「移除」東西的分支只有這幾個：
  - `provider-session.detached`（`orchestrationV2Projection.ts:380-386`）。
  - `thread.deleted`，會清掉整個 thread（`threads.ts:564-575`、`:625-628`）。
  - 可見性變成 false 時，從 `visibleTurnItems` 拿掉（`orchestrationV2Projection.ts:450-452`、`:105-125`）。
- 伺服器的正式程式碼裡沒有任何刪除 projection turn items 或 messages 的 SQL，只有測試工具 `testkit/ProjectionReplay.ts:110-123` 有。
- 手機會把 projection 存進本機快取（`threads.ts:249-290`、`:629-639`），下次用 `afterSequence` 續傳。只要沒觸發 snapshot，被伺服器刪掉的列就會一直留在手機上。
- 新版事件的相容性：舊客戶端遇到未知的事件類型，會解碼成 `unknown-event`，然後跳過（`orchestrationV2.ts:4132-4163`；`threads.ts:534-541`）。所以如果 fork 新增一個 `turn-item.removed` 事件，**沒改過的手機 app 不會壞掉，但也不會移除那則訊息**。

### 7. 若要讓移除反映到手機，可以考慮的方向（全部是推論，沒有驗證過）

| 做法 | 要不要改手機 | 說明 |
| --- | --- | --- |
| 新增 `turn-item.removed` 和 `message.removed` 事件，client-runtime 的 reducer 跟著處理 | 要（client-runtime 是手機的相依套件，要重新建置 app） | 最乾淨。舊 app 會忽略這個事件（見第 6 節）。 |
| 只改伺服器，讓訂閱改送 snapshot | 不用 | 手機收到 snapshot 時會整份替換（`threads.ts:655-676`）。不過伺服器只在 `ws.ts:1047-1118` 列出的那幾種情況才送 snapshot。要讓 snapshot 送出去，可以讓即時串流以錯誤結束，觸發客戶端重新訂閱（`threads.ts:1123-1127`），再在續傳判斷裡強制送 snapshot。這是改動既有協定行為的 hack。 |
| 不處理 | 不用 | 伺服器端保持 200 則上限。手機只在冷啟動或快取失效時才和伺服器一致。因為手機預設只看最近一個視窗（第 1 節第 3 點），使用者多半不會察覺。 |

不建議用「把 run 標成 `rolled_back` 來隱藏訊息」（`orchestrationV2Timeline.ts:48-53`）。匯入的訊息 `runId` 是 null，套不上這條規則，而且 rolled_back 還有 checkpoint 等其他語意。
