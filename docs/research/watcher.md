# T3 伺服器有沒有可以重用的檔案監看機制

Ticket：https://github.com/AI-Reletive-Project/t3code/issues/6
程式依據：本機 clone `本機 clone`（main，commit 629d5083b）。下文的 `apps/...`、`.repos/...` 都是相對這個資料夾的路徑。
調查日期：2026-11-16。

## 結論

**1. T3 Code 伺服器現在有沒有監看檔案的程式可以重用？**

有，但只有「模式」可以重用，沒有現成的通用模組。

- 伺服器全部用 Effect 的 `FileSystem.watch`，沒有用 chokidar 或 `@parcel/watcher`。目前有 6 個地方用它，每個地方各自用 `Stream` 運算子包一層（過濾檔名、debounce、sliding buffer）。沒有共用的 watcher 服務或工具函式。
- 最接近需求的是 `apps/server/src/vcs/GitVcsDriverCore.ts:831-870`。它監看一個只會追加的 trace 檔，每次有事件就讀出新增的部分，再切成一行一行處理。這和「追 `.jsonl` 尾端」是同一種問題。不過它每次都用 `readFileString` 讀**整個檔案**，再依字元位置切出新的部分，所以不適合直接拿來用：本機最大的 `.jsonl` 有 81 MB。
- `apps/server/src/vcs/VcsStatusBroadcaster.ts:1014-1038` 是唯一明確處理 Windows 問題的地方，可以參考：目錄被刪掉後，Windows 上的 watch 會瘋狂報事件，所以用 `takeUntilEffect` 停掉；另外用 `Stream.buffer({capacity:1, strategy:"sliding"})` 合併連續事件。
- `apps/server/src/project/AgentSessionScanner.ts:1392-1460` 已經會掃描 `~/.claude/projects` 找出 transcript，但它只做一次性的列目錄和 stat，沒有監看。補掃（rescan）時可以重用它的路徑規則。
- 現有用法全部都是**非遞迴**監看單一目錄，沒有人監看過大型目錄樹。

**2. 在 Windows 上監看 `~/.claude/projects` 有哪些已知問題？**

- **漏事件（最嚴重）**：Node 22.x 和 24.x 內附的 libuv 在 Windows 上給 `ReadDirectoryChangesW` 的緩衝區只有 **4096 bytes**。緩衝區滿了，系統會把整批變更丟掉。這時 libuv 只會回報一個檔名是 NULL 的 `change` 事件。Effect 的 Node 實作遇到檔名是空的事件會直接 `return`，所以**漏了事件也完全不會知道**。另外，Windows 官方文件說，檔案大小和最後寫入時間的變更「要等快取寫回磁碟才偵測得到」，所以追加寫入的通知可能延遲。
- **重複事件**：一次追加寫入可能同時改到大小、最後寫入時間、最後存取時間。libuv 監聽了這幾種變更，而且每一筆都轉成 `change` 事件，所以同一次寫入可能收到好幾個事件（這是推論）。T3 Code 自己的程式註解也寫了「存檔一次會發出好幾個事件」，所以每個用到的地方都有加 debounce。
- **效能**：本機規模是 1,876 個檔案、519 個目錄、1.6 GB。在 Windows 上，遞迴監看是一個系統原生的 handle，所以監看本身不貴；問題在於事件太多（`tool-results/` 的圖片、subagents 等）時，4 KB 緩衝區很容易滿。另一個成本是 Effect 處理每一個 `rename` 事件都會額外做一次 `stat`。
- **其他**：監看中的目錄被移動或改名時，Windows 不會再發出事件；監看中的目錄被刪除時會報 `EPERM`。Effect 的 watch 預設**不遞迴**，要明確傳入 `{ recursive: true }`。

**建議方向（推論）**：用 `FileSystem.watch(projectsDir, { recursive: true })` 當「提示」，只看 `.jsonl` 結尾的事件，依檔案 debounce。每個檔案記住已處理到的 byte offset，用 `open` 加指定位置的 `read` 只讀新增的部分。另外加上低頻率的定時補掃（stat 比對 size 和 mtime），補回緩衝區溢位或延遲造成的漏網事件。也可以自己寫一個 `WatchBackend`，或直接呼叫 `node:fs` 的 watch，這樣才收得到「檔名是 NULL」的溢位訊號，收到時立刻補掃。

---

## 細節

### A. T3 Code 伺服器的 watcher 用法

#### A1. 底層：Effect `FileSystem.watch`，透過 `NodeServices.layer`

- 伺服器的 runtime layer 用 `NodeServices.layer`：`apps/server/src/server.ts:353`、`apps/server/src/server.ts:401`。
- 版本：`pnpm-workspace.yaml:51-52`（`@effect/platform-node` 和 `@effect/platform-node-shared` 都是 4.1.2）、`pnpm-workspace.yaml:71`（`effect` 4.1.3）。
- repo 對 `@effect/platform-node-shared@4.1.2` 有一個 patch（`pnpm-workspace.yaml:272-274`），內容只改 `NodeChildProcessSpawner`（`patches/@effect__platform-node-shared@4.1.2.patch:1-94`），**沒有改到 watch**。
- 在 `apps/`、`packages/` 裡搜尋 `WatchBackend`，結果是零筆。所以伺服器用的是 Effect 預設的 `node:fs.watch` 實作，沒有換成別的 backend。
- `.repos/effect` 是 Effect 上游原始碼的 clone，版本是 4.1.2（`.repos/effect/packages/platform/node-shared/package.json:4`），和 catalog 一致。下面引用的 Effect 原始碼都來自這份 clone。

#### A2. 伺服器裡用到 watch 的地方（不含測試）

| 檔案:行號 | 監看對象 | 包裝方式 |
|---|---|---|
| `apps/server/src/keybindings.ts:606-622` | keybindings 設定檔所在目錄 | 用檔名過濾，再 `Stream.debounce(100ms)`。註解：「Saves emit multiple events; fs.watch may fire before content is flushed.」 |
| `apps/server/src/serverSettings.ts:1272-1342` | settings 檔所在目錄，加上 symlink 目標的目錄 | 用檔名過濾，加上 `Stream.switchMap` 跟著 symlink 換目標，再 `debounce(100ms)` |
| `apps/server/src/environmentTheme.ts:275-283` | theme 目錄 | `debounce(100ms)`，觸發後整個目錄重讀 |
| `apps/server/src/provider/Drivers/CodexDriver.ts:404-413` | Codex 帳號目錄裡的 `auth.json` | 用檔名過濾，再 `debounce(250ms)`；目錄不存在（`NotFound`）時忽略 |
| `apps/server/src/vcs/VcsStatusBroadcaster.ts:1014-1038` | `.git` 裡 HEAD 所在的目錄 | `onReady`、目錄刪除時停止（Windows 專用）、sliding buffer |
| `apps/server/src/vcs/GitVcsDriverCore.ts:831-870` | git trace2 檔（只會追加） | 每個事件都讀整個檔案，依字元位置取出新的部分，切行處理 |

伺服器以外的地方也有直接用 `node:fs` watch 的：`infra/task-cache/src/telemetry.ts:145-148`（`recursive: true`，出錯就關掉）、`apps/desktop/src/app/DesktopAppActivation.ts:247-259`。

#### A3. 和 Windows 有關的現有處理

- `apps/server/src/vcs/VcsStatusBroadcaster.ts:1024`：註解寫「On Windows a watch outlives its deleted directory and reports it ~150k times a second.」程式在收到 `Remove` 事件時檢查目錄還在不在，不在就用 `takeUntilEffect` 結束 stream。
- `apps/server/src/vcs/VcsStatusBroadcaster.ts:1018`：Git 是用把 `HEAD.lock` 改名成 `HEAD` 的方式更新 HEAD，所以監看的是目錄，不是檔案本身。
- 其他地方沒有針對 Windows 的特別處理。

#### A4. 可以重用的「追尾端」寫法與它的限制

`apps/server/src/vcs/GitVcsDriverCore.ts:831-857`：

- 用 `Semaphore` 確保同一時間只有一個讀取在跑。
- 用 `Ref` 記住 `processedChars` 和還沒湊滿一行的 `remainder`，每次只處理新增的完整行。最後再處理收尾的那一行（`:872-884`）。
- 限制：每次都用 `fs.readFileString(traceFilePath)` 讀**整個檔案**（`:833`）。本機 `.jsonl` 大小的中位數是 0.8 MB，p90 是 9 MB，最大 81 MB（見 C 節）。每收到一個事件就全檔重讀，成本太高。改成用 `fs.open` 搭配 `read`（指定位置）、以 byte offset 讀取，比較合適（推論）。

#### A5. 現有的 `~/.claude/projects` 掃描

`apps/server/src/project/AgentSessionScanner.ts:1392-1460` 的 `discoverClaudeTranscripts` 會列出 `projects/<專案>/*.jsonl`，並且有操作次數上限（budget），還會先看最近修改的資料夾。它可以當補掃的基礎，但它沒有任何監看功能。

#### A6. 相依套件

- `apps/server/package.json:38-80`：沒有 chokidar，也沒有 `@parcel/watcher`。
- `pnpm-lock.yaml` 裡的 chokidar 4.0.3 和 5.0.0 都是間接相依：`@alchemy.run/node-utils`（`pnpm-lock.yaml:13202-13204`）、`@astrojs/check`（`:13273-13276`）、`@tanstack/router-plugin`（約 `:17055-17067`）、`unstorage`（約 `:23652-23655`）。整個 lockfile 沒有 `@parcel/watcher`。
- 結論：如果要改用 chokidar 或 `@parcel/watcher`，得新增直接相依。

### B. Effect `FileSystem.watch` 的實作細節（4.1.2）

原始碼：`.repos/effect/packages/platform/node-shared/src/NodeFileSystem.ts:660-721`。

- 底層直接呼叫 `NFS.watch(path, { recursive: options?.recursive ?? false }, ...)`（`:667-669`），所以**預設不遞迴**。介面文件也是這樣寫：`.repos/effect/packages/effect/src/FileSystem.ts:406-415`、`:1138-1149`。CHANGELOG 有記錄恢復 `recursive` 選項、預設不遞迴：`.repos/effect/packages/platform/node-shared/CHANGELOG.md:465`。
- **檔名是空的事件會被丟掉**：`if (!path) return`（`NodeFileSystem.ts:670`）。下一節會說明，Windows 緩衝區溢位時 libuv 回報的正好是檔名為 NULL 的 `change`，所以 Effect 這層收不到溢位訊號。
- `rename` 事件：每一個都 `Effect.runFork` 一次 `stat`，檔案存在就當成 `Create`，不存在就當成 `Remove`（`:672-677`）。這有兩個後果（推論）：一是大量事件時會多出很多 stat；二是 stat 是非同步的，`Create` 和 `Remove` 送進佇列的順序可能和實際發生的順序不同。`change` 事件則直接當成 `Update`（`:679-681`）。
- 佇列：用 `Stream.callback` 建立，預設是無上限的緩衝（`.repos/effect/packages/effect/src/Stream.ts:698`、`:725-731`）。下游處理太慢時，記憶體會一直長，但不會丟事件。
- watcher 發生錯誤時，stream 會以 `PlatformError` 失敗（`NodeFileSystem.ts:685-698`）；watcher 被關閉時 stream 正常結束（`:699-701`）。呼叫端要自己決定要不要重試。
- 可以替換 backend：如果環境裡有提供 `FileSystem.WatchBackend`，就優先用它（`:708-721`；測試在 `.repos/effect/packages/platform/node-shared/test/NodeFileSystem.test.ts:148-182`）。T3 Code 可以自己寫一個 backend 把溢位訊號傳出來，但這個機制是整個 runtime 共用的，會影響其他 watcher（推論）。
- 遞迴監看在測試裡驗證過：`NodeFileSystem.test.ts:184-199`。

### C. 本機 `~/.claude/projects` 的規模（只看數量和結構，沒有讀內容）

測量路徑：`%USERPROFILE%\.claude\projects\`。

- 總大小 1.6 GB。
- 目錄：第一層（專案）86 個，第二層 298 個，第三層 135 個，共 519 個。第三層的目錄名稱是 `tool-results`（90 個）和 `subagents`（45 個）。
- 檔案共 1,876 個：`.jsonl` 488、`.json` 505、`.jpg` 343、`.md` 253、`.png` 157、`.txt` 127，其他少量。
- 直接放在專案目錄下的 session `.jsonl` 有 296 個。其餘的 `.jsonl` 在更深的目錄裡（應該是 `subagents/`，推論）。
- `.jsonl` 大小：中位數 812,109 bytes，p90 9,036,322 bytes，最大 81,439,353 bytes。
- 最深是 `projects/<專案>/<session>/<子目錄>/<檔案>`，共 4 層。
- 推論：真正要同步的檔案只有第一層的 `<session-id>.jsonl`。但遞迴監看會一併收到 `tool-results`、圖片、`subagents` 的事件，這些都在搶同一個 4 KB 緩衝區。也可以考慮對每個專案目錄各開一個非遞迴監看（86 個 handle），再加一個非遞迴監看看 `projects` 本身有沒有新增專案目錄，這樣可以減少雜訊。

### D. Node `fs.watch` 在 Windows 的已知問題

#### D1. 官方文件（https://nodejs.org/api/fs.html#caveats）

- 「The `fs.watch` API is not 100% consistent across platforms」。
- Windows：監看的目錄被移動或改名時，不會再發出事件；監看的目錄被刪除時會報 `EPERM`。
- Windows 的實作依賴 `ReadDirectoryChangesW`（https://nodejs.org/api/fs.html#availability）。在網路磁碟和虛擬化環境上可能不可靠。
- 回呼的 `filename` 參數「not always guaranteed to be provided」，文件要求呼叫端準備好 `null` 時的處理方式（https://nodejs.org/api/fs.html#filename-argument）。
- `recursive`：只在支援的平台有效。根據 history 表，v19.1.0 才加入 Linux、AIX、IBM i 的支援；Windows 和 macOS 原本就支援（https://nodejs.org/api/fs.html#fswatchfilename-options-listener）。
- `fs.watchFile`（stat 輪詢）文件說它比 `fs.watch`「slower and less reliable」，而且要自己比對 `mtimeMs`（https://nodejs.org/api/fs.html#fswatchfilefilename-options-listener）。

#### D2. libuv 的 Windows 實作（Node 實際用的程式）

- Node v22.x 和 v24.x 內附的 libuv：`const unsigned int uv_directory_watcher_buffer_size = 4096;`。遞迴和非遞迴都用這個大小
  （https://raw.githubusercontent.com/nodejs/node/v22.x/deps/uv/src/win/fs-event.c 、https://raw.githubusercontent.com/nodejs/node/v24.x/deps/uv/src/win/fs-event.c ）。
- libuv 上游的 v1.x 分支已經改成遞迴監看用 64 KB、非遞迴用 4 KB（https://raw.githubusercontent.com/libuv/libuv/v1.x/src/win/fs-event.c ）。哪一個 Node 版本會帶入這個改動，我**沒有查到**。T3 Code 要求的 Node 版本是 `^22.16 || ^23.11 || >=24.10`（`apps/server/package.json:104-106`），目前應該當作 4 KB 處理。
- 監聽的變更種類：`FILE_NAME | DIR_NAME | ATTRIBUTES | SIZE | LAST_WRITE | LAST_ACCESS | CREATION | SECURITY`。
- 對應方式：`ADDED`、`REMOVED`、`RENAMED_OLD_NAME`、`RENAMED_NEW_NAME` 都變成 `rename`；`MODIFIED` 變成 `change`。
- 系統回傳 0 bytes 時：`handle->cb(handle, NULL, UV_CHANGE, 0)`，也就是一個檔名是 NULL 的 `change` 事件。
- 8.3 短檔名：會嘗試用 `GetLongPathNameW` 轉成長檔名，失敗時可能直接回報短檔名（同一個上游檔案）。

#### D3. Windows API 本身（https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-readdirectorychangesw）

- 緩衝區溢位：函式仍然回傳成功，但「the entire contents of the buffer are discarded」，`lpBytesReturned` 是 0。官方建議「compute the changes by enumerating the directory or subtree」。
- `ERROR_NOTIFY_ENUM_DIR`：系統無法記錄所有變更，同樣要重新列目錄。
- `FILE_NOTIFY_CHANGE_SIZE` 和 `FILE_NOTIFY_CHANGE_LAST_WRITE`：「detected only when the file is written to the disk … detection occurs only when the cache is sufficiently flushed」。對「同一個 handle 一直開著、持續追加寫入」的檔案，通知可能延遲（推論：要看 Claude Code 每次寫入後會不會關檔或 flush，這點我沒有驗證）。
- 監看網路路徑時，緩衝區超過 64 KB 會失敗。

#### D4. 綜合判斷

- **漏事件**：事件一多，4 KB 緩衝區就會溢位，整批事件被丟掉；Effect 又把溢位訊號丟掉。結論是一定要有補掃機制，不能只靠 watch。
- **重複事件**：一次追加可能同時改到 SIZE、LAST_WRITE、LAST_ACCESS，而 libuv 對每一筆 `FILE_ACTION_MODIFIED` 都發一個 `change`，所以一次寫入可能收到多個事件（推論）。T3 Code 現有的每個用法都加了 debounce，也印證這一點（`apps/server/src/keybindings.ts:606`）。同步邏輯本身必須是冪等的：用 byte offset 判斷，重讀同一段不會產生重複訊息。
- **效能**：在 Windows 上遞迴監看只用一個系統 handle，對 1,876 個檔案的規模來說，監看本身不是瓶頸（推論）。真正的成本在事件處理：Effect 每個 `rename` 都做一次 stat，再加上讀檔的方式（不要全檔重讀）。

### E. 其他套件（參考用，目前都不在相依裡）

- **chokidar**（https://github.com/paulmillr/chokidar）：README 說預設用原生 `fs.watch`；`auto` 模式在 macOS 和 Windows 用原生遞迴監看。另外有 `polling` backend（舊選項 `usePolling` 是它的別名），適合網路磁碟，但比較耗 CPU。`awaitWriteFinish` 可以避免寫到一半就觸發。v5 只支援 ESM、需要 Node 20 以上；v6 需要 Node 22.22 以上。因為它在 Windows 上同樣建在 `fs.watch` 上，4 KB 緩衝區的問題應該一樣存在（推論，沒有驗證它有沒有額外的溢位補償）。
- **@parcel/watcher**（https://github.com/parcel-bundler/watcher ，`src/windows/WindowsBackend.cc`）：原生模組，自己呼叫 `ReadDirectoryChangesW`，緩衝區用 `DEFAULT_BUF_SIZE 1024 * 1024`（1 MB）；網路路徑出錯時退回 64 KB。溢位（`ERROR_NOTIFY_ENUM_DIR`）時會丟出錯誤「Buffer overflow. Some events may have been lost.」，**不會自動補掃**，但至少讓呼叫端知道有漏。代價是多一個原生相依，打包到桌面版和 exe 都要處理（推論）。
- 「`fs.watch` 一次寫入觸發兩次事件」的社群回報很多，但大多是 2012 到 2016 年的舊資料。例如 https://github.com/nodejs/node/issues/6112 已經關閉、沒有修正。這部分我沒有找到 Node 官方的最新結論。

### F. 給後續 ticket 的建議設計（全部是推論）

1. 監看方式：`fs.watch(projectsDir, { recursive: true })`，只看 `.jsonl` 結尾、而且路徑剛好是 `<專案>/<session>.jsonl` 兩層的事件，其他忽略。
2. 每個檔案做 debounce（例如 100 到 250 ms，和現有用法一致），合併方式參考 `VcsStatusBroadcaster` 的 sliding buffer。
3. 讀取方式：每個檔案記住 byte offset 和還沒湊滿一行的部分，用 `open` 加 `read`（指定位置）只讀新增的段落。發現檔案變小（被截斷或換成新檔）時，從頭重讀。
4. 補掃：用 `AgentSessionScanner` 的列舉方式定期 stat（例如每 5 到 10 秒），比對 size 和 mtime，補回溢位或快取延遲造成的漏網事件。
5. 如果需要立即知道有溢位：不要經過 Effect 的 `FileSystem.watch`，直接用 `node:fs` 的 `watch`，收到 `filename === null` 就馬上補掃。
6. 目錄被刪除時：沿用 `VcsStatusBroadcaster.ts:1024-1029` 的做法，避免 Windows 上事件狂刷。
