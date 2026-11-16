# Claude 紀錄檔的格式與寫入時機

Ticket：<https://github.com/AI-Reletive-Project/t3code/issues/4>
調查日期：2026-11-16。依據：T3 Code `本機 clone`（main，629d5083b）的 `apps/server/src/project/AgentSessionScanner.ts`，以及本機 `%USERPROFILE%\.claude\projects\` 底下的實際紀錄檔（只看結構，不抄文字內容）。

## 結論

1. **看得到的訊息**：`type` 是 `user` 或 `assistant`，而且 `isSidechain`、`isMeta`、`isCompactSummary` 都不是 `true`，`message.content` 裡有非空的 `text` 區塊（或 `content` 本身是非空字串）。T3 Code 現有的篩選就是這個規則（`AgentSessionScanner.ts:342-344`、`594-606`、`701-711`）。其他紀錄全部是內部紀錄，要略過：`api-request`、`api-request-blob`、`api-request-shape`、`file-history-snapshot`、`file-history-delta`、`attachment`、`system`（含 `compact_boundary`、`stop_hook_summary`）、`queue-operation`、`last-prompt`、`custom-title`、`ai-title`、`agent-name`、`mode`、`permission-mode`、`relocated`、`atis-latch`、`tools-used`、`cost-state`。`user` 裡的 `tool_result`、`assistant` 裡的 `thinking` / `tool_use` 也不是文字訊息。
2. **sidechain**：在現在的版本，subagent 的對話不寫在主檔，而是寫在 `<sessionId>/subagents/agent-*.jsonl`，每一筆都是 `isSidechain: true`；主檔裡的 `isSidechain` 全是 `false`。T3 Code 只掃專案資料夾第一層的 `.jsonl`，所以根本不會讀到 subagent 檔（`AgentSessionScanner.ts:1441-1444`）。
3. **寫入時機**：一則助理回答（一次 API 回應，同一個 `message.id`）**不是串流途中逐字寫入**，而是**整個回應結束後才一次寫進檔案**（觀察到約 0.1–0.5 秒後），然後才執行工具。證據：實測時文字區塊完成後 8.5 秒內檔案都沒有它，直到最後一個區塊結束才三筆一起出現；而且同一回應的每一筆都已經帶最終的 `stop_reason` 和 `usage.output_tokens`。使用者訊息在送出、開始處理時寫入。
4. **拆筆與改寫**：一則回應**會被拆成多筆**，每筆 `assistant` 紀錄剛好只有一個 content 區塊（`thinking`、`text`、`tool_use` 各一筆），用同一個 `message.id`、遞增的 `apiBlockIndex` 串起來；同一組中間可能夾著 `tool_result` 等其他紀錄。一個 Claude「回合」（使用者送出到 `end_turn`）會有很多次 API 回應，所以會有多個 `message.id`，其中可能有好幾筆 `text`。**沒有觀察到改寫**：檔案只往後追加，三個主檔都沒有重複的 `uuid`，在觀察期間檔案大小只增不減，也沒看到半行。
5. **壓縮之後的 `parentUuid` 鏈**：壓縮時追加一筆 `type: "system", subtype: "compact_boundary"`，它的 `parentUuid` 是 `null`（鏈在這裡斷開），另有 `logicalParentUuid` 指向壓縮前最後一筆紀錄。緊接著是一筆 `isCompactSummary: true`（同時 `isVisibleInTranscriptOnly: true`）的 `user` 摘要，`parentUuid` 指向 boundary；之後的新紀錄接在摘要後面。壓縮前的紀錄**留在檔案裡不刪不改**，只是從檔尾沿 `parentUuid` 往回走會停在 boundary，走不到它們。`compactMetadata.preservedSegment` / `preservedMessages` 記錄壓縮時保留下來的幾筆舊紀錄 uuid（它們在檔案裡仍在原位置，沒有重寫）。
6. **對即時同步的含意（推論）**：照檔案順序增量讀，只取完整的行（以 `\n` 結尾），套用 T3 Code 現有的篩選規則，就能得到看得到的文字訊息；不需要沿 `parentUuid` 走，也不需要處理改寫。要用 `uuid` 去重（萬一重讀）。同一個 `message.id` 的多筆 `text` 若想合併成一則訊息，要自己依 `message.id` 合併；T3 Code 現在的匯入是一筆 `text` 一則訊息。

## 細節

### 1. 觀察方法

- 結構統計：在 scratchpad 寫 Python 腳本（只印 `type`、欄位名稱、content 區塊型別、uuid 關係、時間戳記、筆數，不印文字），對三個主檔執行：
  - `<專案資料夾>\<工作階段甲>-…jsonl`（進行中，987 筆，`entrypoint: claude-desktop`，`version 2.1.325`）
  - `<專案資料夾>\<工作階段乙>-…jsonl`（1530 筆，有一次手動 `/compact`，`version 2.1.308–2.1.325`）
  - `<另一個專案資料夾>\<工作階段丙>-…jsonl`（891 筆，前 36 筆 `entrypoint: cli`（2.1.243），之後 `claude-desktop`（2.1.246））
- 寫入時機實測：在背景每 0.1 秒輪詢本調查自己的 subagent 紀錄檔（`<工作階段甲>-…\subagents\agent-<id>.jsonl`）的檔案大小，有新行就記錄「出現在磁碟的時間」和該筆的 `timestamp`、`message.id`、`stop_reason`、`usage.output_tokens`。同時刻意產生「一段文字 + 快速工具呼叫 + 長串流工具呼叫」的回應。

### 2. 紀錄類型

`<工作階段甲>` 的統計（`<工作階段乙>` 與 `<工作階段丙>` 類似）：

| type | 筆數 | 有 `uuid`/`parentUuid` | 說明 |
|---|---|---|---|
| `assistant` | 220 | 是 | 每筆一個 content 區塊：`tool_use` 113、`thinking` 62、`text` 45 |
| `user` | 144 | 是 | `tool_result` 113、字串 26（其中 4 筆 `isMeta`）、`text` 區塊 4（皆 `isMeta`）、`image` 1 |
| `api-request` | 120 | 否 | 每次 API 請求一筆，`id` 對應 `assistant.requestRef`（220/220 對得上），一定寫在對應回應之前 |
| `api-request-blob` / `api-request-shape` | 39 / 2 | 否 | 請求內容的去重存放（`hash` + `message`）與 system/tools 形狀 |
| `attachment` | 104 | 是 | hook 輸出、環境、skill 清單、日期等注入內容，`attachment.type` 有 20 多種 |
| `system` | 27 | 是 | `subtype`：`stop_hook_summary`；另見 `compact_boundary`、`local_command`、`turn_duration` |
| `file-history-snapshot` / `file-history-delta` | 23 / 1 | 否 | 檔案備份；用 `messageId` 對應，不在 `parentUuid` 鏈上 |
| `queue-operation` | 48 | 否 | `operation`：`enqueue` / `dequeue` / `remove`；助理忙碌時使用者先打的訊息 |
| `last-prompt` | 46 | 否 | `lastPrompt`、`leafUuid`（目前分支末端） |
| `custom-title` / `ai-title` / `agent-name` / `mode` / `permission-mode` / `relocated` / `atis-latch` / `tools-used` / `cost-state` | 數十筆 | 否 | session 層級的狀態，**重複追加**（例如 `custom-title` 48 筆），以最後一筆為準 |

觀察方法：Python 統計 `type`/`subtype` 與欄位名稱。不同 Claude Code 版本的類型集合不同（例如 2.1.24x 的檔案有 `ai-title`、`permission-mode`，沒有 `api-request`；2.1.3xx 有 `api-request`、`custom-title`），所以同步程式**應該用白名單**（只取 `user`/`assistant`），不要用黑名單。

### 3. 看得到的使用者 / 助理訊息

T3 Code 的做法（Claude 部分）：

- 隱藏條件：`isSidechain === true || isMeta === true || isCompactSummary === true`（`AgentSessionScanner.ts:342-344`）。
- 只收 `type` 為 `user` / `assistant`，從 `message.content` 抽文字（`AgentSessionScanner.ts:594-606`）；抽文字時只接受區塊 `type` 為 `text` / `input_text` / `output_text`，字串 content 直接用，多個文字區塊以換行連接（`AgentSessionScanner.ts:308-322`）。沒有文字的紀錄（`tool_use`、`tool_result`、`thinking`、`image`）被略過。
- 串流讀檔時用同一規則決定要不要保留（`AgentSessionScanner.ts:695-711`），只保留第一則使用者訊息加最近 `MAX_IMPORTED_MESSAGES` 則（`AgentSessionScanner.ts:723-746`）。
- 模型名稱 `"<synthetic>"` 是本機錯誤回應用的假模型，不當作模型（`AgentSessionScanner.ts:370-372`）。
- **不看 `parentUuid`**，完全照檔案順序（`AgentSessionScanner.ts:580-606`）。
- 標題：`customTitle` 優先於 `aiTitle`（`AgentSessionScanner.ts:361-374`、`681`）。

實際檔案中，通過這個篩選但其實不是「人打的字」的情況（觀察方法：只印字串開頭的 XML 標籤名稱與欄位值，不印內容）：

- 使用者在本機執行 slash command（例如 `/compact`）會留下以 `<command-name>`、`<local-command-stdout>` 開頭、**非** `isMeta` 的 `user` 紀錄（`<local-command-caveat>` 那筆才是 `isMeta`）。T3 Code 現在會把它們當成使用者訊息。
- `origin.kind` 有 `"human"` 與 `"peer"`（其他 session 傳來的訊息，`promptSource: "system"`）。真人輸入的 `user` 紀錄帶 `origin`、`promptSource`、`turnOrigin`、`turnPosition`；沒有 `origin` 的非 meta 文字 `user` 紀錄是上面的 command 紀錄或中斷提示之類（**推論**：`<工作階段乙>` 有 2 組回應的 `stop_reason` 為 `null`，後面緊接一筆沒有 `origin` 的文字 `user`，應是使用者中斷時寫入的提示）。
- 若同步只想要真人輸入，可再加條件 `origin.kind === "human"`（**推論**，只在 2.1.3xx 的 desktop 檔案觀察到這組欄位）。

### 4. 一則助理回答的拆法

觀察方法：依 `message.id` 分組統計。

- 每筆 `assistant` 紀錄的 `message.content` 長度都是 1（三個檔案 825 筆全部如此）。
- 同一 `message.id` 有 1–9 筆；`apiBlockIndex` 依序 0、1、2…；同組每筆的 `stop_reason`、`usage` 完全相同（`<工作階段甲>` 120/120 組、`<工作階段乙>` 137/137 組），都是最終值。
- 同組不一定連續：`<工作階段甲>` 有 12 組、`<工作階段乙>` 有 28 組中間夾了其他紀錄（例如 `tool_use` → 它的 `tool_result` → 同回應的下一個 `tool_use`）。
- 每組最多出現 1 個 `text` 區塊（本機資料），但一個回合有多組，所以一個回合可能有好幾筆 `text`。
- 同組各筆的 `timestamp` 是各區塊完成串流的時間（例如兩個 `tool_use` 相差 10.65 秒，而後者的 `toolInputStreamMs` 正好是 10644），不是寫入時間。

### 5. 寫入時機（實測）

測試 A（三個區塊：文字 → 快速的 Bash → 長串流的 Write）：

| 紀錄 | `timestamp` | 出現在磁碟 |
|---|---|---|
| `assistant` `text` | 11:55:13.820 | 11:55:22.334 |
| `assistant` `tool_use`（Bash） | 11:55:14.732 | 11:55:22.334 |
| `assistant` `tool_use`（Write） | 11:55:20.974 | 11:55:22.334 |
| `user` `tool_result` ×2、`api-request` | 11:55:22.32–22.36 | 11:55:22.445 |

文字區塊完成 8.5 秒後才出現在檔案，而且和最後一個區塊同時出現；Bash 實際執行時間（工具內印出的時間）也是 22.29，在回應結束之後。測試 B 的兩則回應也一樣：各區塊都在該回應最後一個區塊結束後約 0.1–0.15 秒才一起出現。整個過程沒有看到半行（輪詢讀到的資料都以 `\n` 結尾）。

結論：**同一個 API 回應的所有 `assistant` 紀錄，在回應完成後一次追加**；接著才是工具結果、下一個 `api-request`。所以「幾秒內同步」的延遲下限是「一個 API 回應的長度」，不是逐字串流。中斷的回應也會寫入，只是 `stop_reason` 為 `null`（`<工作階段乙>` 有 2 組）。

限制：
- 這是 2.1.325 desktop 版、subagent 檔的實測；CLI 與主檔的寫入路徑**推論**相同（主檔同樣有「同組 usage 都是最終值」的現象），沒有另外實測。
- 測試中有一次，回應裡的一段文字輸出在 subagent 檔裡找不到對應的 `text` 紀錄（該回應只記了兩筆 `thinking` 和一筆 `tool_use`）；原因不明，只發生在 subagent 檔，主檔的回答沒有觀察到這種情況。
- `timestamp` 不是單調遞增：三個檔案各有 14–35 次比前一筆早（例如 `/compact` 指令的紀錄在 boundary 之後才寫，但時間戳記較早）。同步請用**檔案順序**，不要用 `timestamp` 排序。

### 6. `parentUuid` 鏈

觀察方法：建立 `uuid → 行號` 對照，檢查 `parentUuid` 指向、分支、從檔尾往回走的長度。

- 只有帶 `uuid` 的類型（`user`、`assistant`、`attachment`、`system`）在鏈上；`api-request`、`file-history-*`、各種 title/mode 紀錄沒有 `uuid`。
- `parentUuid` 永遠指向檔案中更早的紀錄（向前指 0 筆）；第一筆有 `uuid` 的紀錄 `parentUuid: null`。
- 正常就有分支：平行工具呼叫時，`tool_use` A 有兩個子節點——同回應的下一個 `tool_use` B 和 A 的 `tool_result`（`<工作階段甲>` 7 處、`<工作階段乙>` 59 處都是這個形狀）。所以鏈不是單純一條線，「父節點 = 上一行」只有約 66%。
- `last-prompt.leafUuid` 指向目前分支的末端。
- `<工作階段丙>`（CLI 開始、desktop 續用）有 7 筆 `parentUuid` 指到檔案裡不存在的 uuid（**推論**：與 resume / 換程式有關，未深入）。

### 7. 壓縮（`<工作階段乙>` 第 785–800 行的結構）

```
785  system        （壓縮前最後一筆）
786–792 agent-name / mode / atis-latch …（無 uuid）
793  system  subtype=compact_boundary  parentUuid=null  logicalParentUuid=<785 的 uuid>
         compactMetadata = { trigger:"manual", preTokens, postTokens, durationMs,
                             cumulativeDroppedTokens, preCompactDiscoveredTools,
                             preservedSegment:{headUuid, anchorUuid, tailUuid},
                             preservedMessages:{anchorUuid, uuids[], allUuids[]} }
794  user  isCompactSummary=true  isVisibleInTranscriptOnly=true  parent=793  content="<文字>"
795  user  isMeta=true  parent=794   （<local-command-caveat>）
796  user  parent=795                （<command-name>）
797  user  parent=796                （<local-command-stdout>）
798… attachment … 之後的新對話接下去
```

- `preservedSegment.headUuid` / `tailUuid` 指向第 783、785 行（壓縮前最後幾筆），`anchorUuid` 指向第 794 行的摘要；這些舊紀錄**沒有被複製或改寫**（全檔無重複 uuid）。`preservedMessages.allUuids` 裡有 4 個 uuid 不在檔案中。**推論**：Claude Code 載入時把保留段接在摘要後面，檔案本身不改。
- 從檔尾沿 `parentUuid` 往回走 351 步就停在 boundary；壓縮前 35 筆看得到的文字紀錄不在這條鏈上。
- T3 Code 因為照檔案順序讀、又隱藏 `isCompactSummary`，所以壓縮前後的訊息都會匯入、摘要不會出現（`AgentSessionScanner.ts:342-344`、`594-606`；測試 `AgentSessionScanner.test.ts:3228-3238`、`3269-3274`）。但 boundary 之後那幾筆 `<command-name>` / `<local-command-stdout>` 會被當成使用者訊息。

### 8. T3 Code 讀檔的相關行為

- 只掃 `~/.claude/projects/<專案>/` 第一層的 `*.jsonl`（`AgentSessionScanner.ts:1392-1469`，特別是 `1441-1444`）；`subagents/`、`tool-results/` 子資料夾不讀。
- 一次讀到 stat 時的檔案大小，讀完再比對檔案身分，檔案在讀的途中變了就放棄這次結果（`AgentSessionScanner.ts:1225-1236`、`1336`）。對持續寫入中的檔案，匯入可能因此失敗、要重試（**推論**：`transcriptIdentity` 的內容未細看）。
- 無法解析的行直接丟掉，不報錯（`AgentSessionScanner.ts:1264-1269`、`459`）；所以讀到檔尾的半行也只是被略過。
- Schema 只宣告了需要的欄位（`AgentSessionScanner.ts:123-152`），沒有 `uuid`、`parentUuid`、`message.id`；增量同步若要去重或合併，需要擴充這個 schema。

### 9. 給增量同步的建議（推論）

1. 記住每個檔案已處理到的 byte offset；檔案變大時從 offset 讀到最後一個 `\n`，剩下的半行留到下次。
2. 每行解析後：只留 `type` 為 `user`/`assistant`、三個隱藏旗標都不是 `true`、有非空文字的紀錄（沿用 `isHiddenClaudeRecord` 與 `extractText`）。視需要再排除 `<command-name>` / `<local-command-*>` 開頭的 `user` 字串，或只收 `origin.kind === "human"`。
3. 用 `uuid` 去重；照檔案順序追加，不用 `timestamp` 排序，不用沿 `parentUuid` 走。
4. 遇到 `compact_boundary` 不用特別處理（摘要已被 `isCompactSummary` 濾掉）；遇到 `parentUuid` 分支（rewind／編輯重送）時，檔案順序會把兩條分支都讀進來——本次資料中沒有這種情況，未驗證。
5. 不要讀 `subagents/` 底下的檔案。
6. 預期延遲 = 該次 API 回應的長度 + 輪詢間隔；一個回合中間的 `text`（工具呼叫前的說明）會在該次回應結束時先出現。
