# 怎麼判斷外部的 Claude 工作階段還在執行（ticket #5）

Ticket：https://github.com/AI-Reletive-Project/t3code/issues/5
調查日期：2026-11-16。本機 Claude Code 版本 2.1.325（Claude 桌面 app 內附，`%APPDATA%\Claude\claude-code\2.1.325\...\claude.exe`）。T3 Code 依據為本機 clone `本機 clone`（main，未執行任何 git 指令，行號以讀取當下為準）。

## 結論

1. **有沒有可靠的方法判斷某個工作階段正被別的程式使用？有，而且有官方介面。**
   - 首選：`claude agents --json`。官方文件說它列出「every live session」，每筆有 `sessionId`、`kind`（`interactive` / `background`）、`pid`，以及行程還活著時的 `status`（`busy` / `waiting` / `idle`）。依據：https://code.claude.com/docs/en/agent-view 、https://code.claude.com/docs/en/cli-reference（`claude agents` 一列）。
   - 底層資料：`~/.claude/sessions/<pid>.json`，一個執行中的 Claude Code 行程一個檔。官方文件說它「holds one small file per running session, used to detect concurrent sessions and crashes」，行程結束時刪除，當機殘留在下次啟動時清掉。依據：https://code.claude.com/docs/en/claude-directory（Application data 段）。檔內有 `pid`、`sessionId`、`procStart`、`status`、`kind`、`entrypoint` 等欄位（本機實測，見細節）。官方沒有說這個檔的格式是穩定介面；同一份 agent-view 文件對 `~/.claude/jobs/<id>/` 明說「not a stable interface」並建議改用 `claude agents --json`。
   - **沒有鎖檔**：transcript（`~/.claude/projects/<project>/<session-id>.jsonl`）不上鎖。官方文件明說兩個終端機同時 resume 同一個 session 會「both write to one transcript」。依據：https://code.claude.com/docs/en/sessions（Branch a session 段末）。
   - **紀錄檔最後寫入時間不可靠**（推論）：idle 的互動式 session 行程還在、但不寫 jsonl；busy 的 session 在長時間工具呼叫中也可能很久不寫。mtime 只能當輔助訊號。
2. **`Session ID … is already in use` 是誰丟的、根據什麼條件？**
   - 是 **Claude Code CLI 本身**（T3 Code 透過 `@anthropic-ai/claude-agent-sdk` 啟動的 `claude` 行程）在啟動參數檢查時丟出，不是 T3 Code。
   - 條件是：用 `--session-id <id>`（SDK 的 `sessionId` 選項）開「新」session，而目前 cwd 對應的 project 目錄下**已經存在** `<id>.jsonl`。它只做 `statSync` 檢查檔案存在，**完全不看有沒有別的行程在用**。依據：本機 claude.exe 2.1.325 內嵌 JS（見細節 2.1）；上游 issue https://github.com/pingdotgg/t3code/issues/15243 中 `--session-id <existing-id>` 失敗、`--resume <same-id>` 成功的實測。
   - 所以這個錯誤**不是**「外部 Claude 還在執行」的訊號，而是「T3 Code 該用 resume 卻用了 sessionId」的 bug。上游已修：`shouldResume` 會參考 `nativeThreadHasTurns` 與 transcript 狀態（`ClaudeAdapterV2.ts:12749-12754`、`:13937-13943`）。
3. **外部 Claude 真的還在執行時，T3 Code 送訊息會怎樣？**（這才是鎖輸入框真正要擋的情況）
   - 外部是**互動式**（`kind: "interactive"`，Claude 桌面 app 屬於這類，`entrypoint: "claude-desktop"`）：CLI **不擋**，`--resume` 會成功，兩邊同時寫同一個 transcript，之後 resume 可能遺失其中一邊的訊息。依據：CLI 程式碼 `Npt`/`ope` 只把非 interactive 的持有者視為阻擋（細節 2.2）；官方 sessions 文件；issue #15243 中 MatthewRamsey 的實測表「Another SDK query() holding it open → ok」。
   - 外部是**背景 session**（`claude --bg`，`kind` 非 interactive）：CLI 會拒絕，錯誤是 `That session is running in the background (<短 id>)…`，T3 Code 只顯示「Provider turn failed」。依據：issue #15243 MatthewRamsey 留言；https://code.claude.com/docs/en/sessions（Resume a running background session）。
   - 因此（推論）：T3 Code 要鎖輸入框，判斷條件應該是「`claude agents --json`（或 `~/.claude/sessions/*.json`）裡有一筆 `sessionId` 等於本 thread 的 native id、`pid` 不是 T3 Code 自己的 SDK 行程、而且行程還活著」，不能靠 `already in use` 這個錯誤。
4. **上游 commit 629d5083b（PR #37610）相關嗎？** 間接相關。它處理的是相反情況：resume 時 CLI 說 `No conversation found with session ID:`（transcript 不見了），改成下一次送出時開新 native session 並帶 handoff。它不判斷 liveness，也不處理「被別人使用中」。依據：https://github.com/pingdotgg/t3code/pull/37610 ；本機 `ClaudeAdapterV2.ts:387`、`:12951-12953`、`:13921-13926`。

## 細節

### 1. T3 Code 端

| 項目 | 依據 | 說明 |
| --- | --- | --- |
| sessionId / resume 二選一 | `apps/server/src/orchestration-v2/Adapters/ClaudeAdapterV2.ts:2228-2230` | `input.resume ? { resume: id } : { sessionId: id }`，沒有第三種模式。 |
| shouldResume 判斷 | 同檔 `:12749-12754` | `resumeSessionAt` 有值、這個行程已開過（`openedNativeThreads`）、或已有持久化的 provider turn（`nativeThreadHasTurns ?? providerTurnOrdinal > 1`）就 resume。註解直接寫明要避免 "already in use"。 |
| 匯入的 session 走 resume | 同檔 `:13915-13943` | `resumeThread` 讀 transcript 狀態，有對話內容就加進 `openedNativeThreads`，註解：「A session imported from the terminal has history but no T3 provider turns; opening it with sessionId fails "already in use".」。`:13918-13919` 註解「A live process holds the conversation even before saving it」指的是 T3 自己的 query，不是外部行程。 |
| 只標記成功的 open | 同檔 `:12827-12835` | 失敗的 open 不會加入 `openedNativeThreads`。 |
| transcript 不見的處理（#37610） | 同檔 `:387`、`:12951-12953`、`:13921-13926` | 偵測 `No conversation found with session ID:`，記進 `missingNativeThreads`，下一次 `resumeThread` 丟錯，讓 `ProviderTurnStartService` 的 fallback 開新 session。 |
| 沒有任何外部 liveness 偵測 | 在 `apps/**/*.ts` 搜尋 `.claude/sessions`、`procStart`、`messagingSocketPath`、`kind.*"bg"` 均無相關結果 | T3 Code 目前不讀 `~/.claude/sessions/`，也不呼叫 `claude agents`。 |

### 2. Claude Code CLI 端（讀本機 claude.exe 2.1.325 內嵌的壓縮 JS，函式名是壓縮後的名字，會隨版本變）

#### 2.1 `already in use` 的條件

在 claude.exe 內找到的啟動檢查（節錄，壓縮碼）：

```js
if(Le){if(!De){let s=Et(Le);if(!s)return kn("Error: Invalid session ID. Must be a valid UUID.");
if(!(o.forkSession&&o.resume===s)&&uxr(s))return kn(`Error: Session ID ${s} is already in use.`,lnr)}}
function uxr(e){if(K())return!1;let n=h5(),r=zg(n,`${e}.jsonl`),s=se();try{return s.statSync(r),!0}catch{return!1}}
function h5(){return RX()??w3e()}
```

- `Le` 是 `--session-id` 的值，`De` 是 `sdkUrl`（遠端模式時不檢查）。
- `uxr` 只檢查「目前 project 目錄（`h5()`）下的 `<id>.jsonl` 是否存在」。
- 例外：`--fork-session` 搭配 `--resume <同一個 id>` 時不檢查。
- 結論：這是「ID 已被某個 transcript 佔用」的檢查，跟行程是否在跑無關。

#### 2.2 resume 時怎麼看「別的行程正在用」

```js
async function ope(e,n){let r=(await Npt(e)).find((s)=>s.kind!=="interactive");if(!r)return null; ...}
async function Npt(e,n){ ... listAllLiveSessions() ...
  for(let o of r){if(o.sessionId?.toLowerCase()!==i||o.pid===process.pid||!o.kind)continue;
  if(o.kind==="interactive"&&o.parkedJobId!==void 0)continue; s.push({kind:o.kind,jobId:o.jobId})} return s}
```

- `ope` 找出持有同一個 sessionId 的活行程，**只有非 interactive 的（背景 session）才算阻擋**。interactive 的持有者不會讓 resume 失敗。
- 這和官方文件、issue #15243 的實測一致。

#### 2.3 「活著」怎麼判斷（`listAllLiveSessions` = 壓縮名 `$Q`）

```js
async function $Q(e,n){let r=await T({...}), ...
  f=r.map((l)=>d(l)||fl(l.pid)),
  g=await Promise.all(r.map((l,x)=>f[x]&&(d(l)||Tk(l.pid,l.procStartFt??l.procStart)))), ...
  for(...){ if(g[l])o.push(k); else if(m&&x4r(k,a)&&wh(k.pid))oe(x,k.pid,a,e) } return o}
```

- 讀 `~/.claude/sessions/*.json` 的紀錄（`T()`）。
- 先看 `pid` 是否存在（`fl`），再比對行程啟動時間與紀錄的 `procStart`（`Tk`），避免 PID 被重複使用造成誤判。
- 行程已死的紀錄會被清掉（`oe`）。
- 另有 `pidDomain` 欄位（本機值形如 `win32:<主機名>`），用來區分不同機器/容器（`ae` 函式：domain 不同就當作 present）。

### 3. 本機 `~/.claude/` 結構（只看檔名與欄位名，未讀對話內容）

- 目錄：`sessions/`、`session-env/`、`projects/`、`jobs/`、`daemon/`、`file-history/`、`shell-snapshots/` 等。
- `sessions/` 內容：每個執行中的 Claude 行程一組 `<pid>.json` 與 `<pid>.<hash>.key`。調查當下有 3 組，3 個 pid 都對得上正在執行的 `claude.exe`（`Get-Process`）。
- `<pid>.json` 欄位名（值只列與判斷有關、不涉隱私者）：
  - `pid`、`sessionId`、`cwd`、`startedAt`（Unix ms）
  - `procStart`：Windows 上是 FILETIME（100ns，自 1601 起）。實測換算後與 `startedAt` 相差約 1 秒，可用來比對行程身分。
  - `version`（`2.1.325`）、`peerProtocol`、`peerFeatures`
  - `kind`：本機 3 筆皆為 `interactive`
  - `entrypoint`：本機 3 筆皆為 `claude-desktop`（桌面 app 開的 session）
  - `hostSessionId`、`pidDomain`、`messagingSocketPath`（Windows named pipe）
  - `name`、`nameSince`、`nameSource`
  - `status`：實測出現 `idle`、`busy`、`waiting`；`statusUpdatedAt`、`updatedAt`
  - `waitingFor`（`status` 為 `waiting` 時出現）
- 推論：桌面 app 開著某個對話時，即使 idle，對應的 `claude.exe` 行程仍在、`sessions/<pid>.json` 仍在。所以「行程存在」代表「被佔用中」，`status` 再區分是否正在跑 turn。

### 4. 官方文件重點

- `claude agents --json`：列出所有 live session；欄位 `cwd`、`kind`、`startedAt` 必有；`pid`、`status`（`busy`/`waiting`/`idle`）只在行程活著時出現；`sessionId` 在有設定時出現。https://code.claude.com/docs/en/agent-view
- `~/.claude/jobs/<id>/` 不是穩定介面，建議用 `claude agents --json`。同上。
- `sessions/`：每個執行中 session 一個小檔，用來偵測並行 session 與當機。https://code.claude.com/docs/en/claude-directory
- 同時 resume 同一 session：兩邊寫同一份 transcript，之後可能遺失訊息，要保留兩邊請加 `--fork-session`。https://code.claude.com/docs/en/sessions
- resume 一個仍在跑的背景 session：v2.1.285 起會改成 `claude attach`；但帶了 `--output-format`、`--permission-mode` 等旗標時（SDK 一定會帶）會 exit 1 並印出 `claude attach <id>` 提示。同上。
- `--session-id`：「Use a specific session ID for the conversation (must be a valid UUID)」，文件沒有說明 already in use 的條件。https://code.claude.com/docs/en/cli-reference

### 5. 上游 issue / PR

- pingdotgg/t3code#15243（CLOSED 2026-11-03）：匯入的 Claude session 第一次 follow-up 用了 `sessionId` 而非 `resume`，CLI 回 `already in use`。多位回報者實測 `--session-id <existing>` exit 1、`--resume <same>` exit 0。修正 PR 提及 #23202、#15376 / #23219。https://github.com/pingdotgg/t3code/issues/15243
- 同 issue 中 MatthewRamsey 的留言：外部是 `claude --bg` 背景 session 時，resume 會被 CLI 拒絕（`That session is running in the background`）。T3 丟掉 stderr，只顯示「Provider turn failed」。他提出的修法之一是「resume 前檢查有沒有 live 的 `kind: "bg"` 持有者」。
- pingdotgg/t3code#37610（MERGED，對應 commit 629d5083b）：處理 transcript 不見（`No conversation found`）時改開新 session。與 liveness 無直接關係。https://github.com/pingdotgg/t3code/pull/37610

## 對規劃的建議（推論）

1. 偵測來源優先順序：
   - `claude agents --json`：官方文件化介面，但要另外起行程，而且「Opening agent view requires an interactive terminal」只限制開 UI，`--json` 不受此限，這點待實測。
   - 直接讀 `~/.claude/sessions/*.json`：快、可用檔案監看，但不是文件化格式，版本升級可能改欄位。
2. 判定「外部佔用中」：
   - 紀錄的 `sessionId` 等於 thread 的 native id。
   - `pid` 存在且 `procStart` 對得上。
   - 該 pid 不是 T3 Code 自己啟動的 SDK 子行程。待實測：T3 的 SDK 行程是否也會寫進 `sessions/`，`entrypoint` 預期會不同。
3. 鎖輸入框的範圍：
   - 互動式外部 session（桌面 app）CLI 不會擋，但同時寫會讓 transcript 互相覆蓋，所以仍建議鎖。
   - 背景 session CLI 一定擋。
4. 不要用 `Session ID … is already in use` 當 liveness 訊號，它只表示 transcript 檔已存在。
5. 不要用 jsonl 的 mtime 當主要依據。
