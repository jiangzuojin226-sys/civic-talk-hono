# 公民意見投票功能：實作筆記與原理教學

這份文件記錄 issue #107（公民意見投票）從研究到完成的完整過程，寫給想搞懂「為什麼這樣做」而不只是「做了什麼」的人。每一節先講**問題**，再講**解法**，最後講**如果不這樣做會怎樣**。

---

## 〇、這個功能要解決什麼

`/about` 的步驟 3 長期掛著「您也可以對其他人的意見進行投票 🚧 功能研發中」。網站上寫了卻做不到的功能，比沒寫還糟——使用者會去找那個按鈕，找不到就會懷疑網站壞了。

功能本身的意義：平台收集了很多公民意見，但**只能讀不能表態**。沒有表態資料，後續的「意見綜整」就只能靠 AI 讀文字猜哪些觀點重要，而不知道有多少人真的同意。

---

## 一、研究階段：我讀了什麼

### 1-1 為什麼要先研究，不直接動手

Bestian 在 issue 回應裡指定「可以參考 pocket-polis 的程式碼」。如果我直接憑想像寫一個投票功能，做出來的 CSV 格式八成跟別人的工具對不上，之後還要重做。

更重要的是：**pol.is 這類審議工具的設計是有累積的**。投票為什麼是三個選項而不是按讚、為什麼要匿名、CSV 為什麼長那樣，背後都有理由。照抄成熟設計比自己重新發明省事也更可靠。

### 1-2 pocket-polis 教了我什麼

我把 <https://github.com/mashbean/pocket-polis> 抓下來，重點讀了兩個檔案：

**`src/conversation.ts`** — 投票的資料結構與寫入邏輯：

```sql
CREATE TABLE votes (
  pid TEXT NOT NULL,        -- 參與者
  sid INTEGER NOT NULL,     -- 意見（statement）
  value INTEGER NOT NULL,   -- 1 同意、-1 不同意、0 略過
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (pid, sid)    -- ← 關鍵：一人一票靠這行
)
```

```js
INSERT INTO votes (...) VALUES (...)
ON CONFLICT(pid, sid) DO UPDATE SET value = excluded.value, updated_at = excluded.updated_at
```

**`src/export.ts`** — pol.is 相容的 CSV 格式：

```
timestamp,datetime,comment-id,author-id,agrees,disagrees,moderated,comment-body
```

這個檔案的註解特別有價值，它記下了幾個「不照做就會壞」的細節：

- `datetime` 欄**不加引號**，`comment-body` 欄**一律加引號**
- 檔案**不能有 BOM**（BOM 是檔案開頭的隱形字元，會讓嚴格比對欄位名稱的程式認不出第一欄）
- 換行必須是 LF（`\n`）不能是 CRLF（`\r\n`）
- `moderated` 欄：`1` 已核准、`0` 未審、`-1` 已拒絕

這些是**別人踩過坑才寫下來的知識**。自己從頭試錯可能要花好幾個小時。

### 1-3 票值為什麼是 1 / -1 / 0

用 `1` 同意、`-1` 不同意，數學上很方便：一串票直接相加，正數代表偏向同意、負數代表偏向不同意。而 `0`（略過）加進去不影響總和，但**有被計數**——這正是「略過」該有的語意：我有參與，但我沒有立場。

如果改成「讚 / 倒讚」兩個選項，看不懂的人只能被迫選邊或乾脆不點，資料就會失真。

---

## 二、資料庫設計

### 2-1 為什麼要新開一張表

有人會想：為什麼不在 `ct_opinions` 加三個欄位 `agrees`、`disagrees`、`passes` 就好？

因為那樣**存不下「誰投的」**。存不下誰投的就做不到：

- 一人一票（不知道你投過沒）
- 改票與收回（不知道要扣哪一票）
- 匿名化的逐票匯出

所以投票必須是獨立的一張表，每一列代表「某人對某則意見的一次表態」。這在資料庫設計裡叫**多對多關聯表**：一個人可以投很多則意見，一則意見可以被很多人投。

### 2-2 複合主鍵：一人一票的真正防線

```sql
PRIMARY KEY (opinion_id, voter_id)
```

**主鍵**的意思是「這張表裡不可能有兩列的這組值一樣」。把 `(意見, 投票者)` 設成主鍵，等於資料庫替你保證了「同一個人對同一則意見只會有一列」。

為什麼不能用程式檢查就好？想像這段程式：

```js
const existing = await 查詢('你投過這則了嗎')
if (!existing) await 寫入(新的一票)
```

看起來沒問題，但如果你手滑連點兩下，兩個請求**幾乎同時**送到伺服器：

```
請求 A：查詢 → 「沒投過」
請求 B：查詢 → 「沒投過」   ← A 還沒寫進去，所以 B 也看到「沒投過」
請求 A：寫入一票
請求 B：寫入一票            ← 變成兩票
```

這叫**競態條件（race condition）**。兩個動作之間的空隙就是漏洞，而這個空隙無論你把程式寫得多快都消不掉。唯一可靠的解法是讓**資料庫**來保證唯一性——因為資料庫的寫入是原子的（不可分割的）。

### 2-3 upsert：改票不用先查

```sql
INSERT INTO ct_opinion_votes (...) VALUES (...)
ON CONFLICT(opinion_id, voter_id) DO UPDATE SET value = excluded.value, updated_at = CURRENT_TIMESTAMP
```

這句的意思是：**先試著新增，如果撞到主鍵就改成更新**。`excluded` 是 SQLite 的特殊關鍵字，指「原本想插入但被擋下來的那筆資料」。

好處是一句 SQL 搞定「第一次投票」和「改票」兩種情況，中間沒有空隙可以插隊。這個動作有個合成詞叫 **upsert**（update + insert）。

### 2-4 提議者的當然贊成票：為什麼「用算的」而不是「存起來」

Bestian 的裁示是：提出意見的人本身就算一票贊成，但不能再另外投票。

有兩種做法：

| 做法 | 優點 | 缺點 |
| --- | --- | --- |
| 投稿時順便寫一筆投票 | 資料直觀 | **既有的幾百則舊意見全都要回填**，回填腳本寫錯就是災難 |
| 讀取時才加上去 | 既有資料一行都不用動 | 每次查詢都要記得加，忘記就會不一致 |

Bestian 指定第二種（「如此也不必對現存的資料作修改」）。缺點的解法是**把這個規則集中在一個地方**——我抽出一個純函式：

```ts
export function toOpinionVoteState(row, viewerId) {
  const agree = Number(row.agrees ?? 0) + (row.author_id !== null ? 1 : 0)
  //                                       ↑ 規則只寫在這一行
  ...
}
```

所有需要票數的地方都呼叫它，就不會有人忘記。CSV 匯出那邊也用同一條規則（程式碼裡有註解互相指認）。

**這是一個通用原則：容易被忘記的規則，要集中到唯一一個地方。**

### 2-5 刪除時要記得清票

意見被刪掉，掛在它身上的票就變成沒有主人的「孤兒資料」。

SQL 有個機制叫 `ON DELETE CASCADE`（連鎖刪除），我也寫了。但這個 repo 的既有程式碼都是**手動逐表刪除**：

```ts
await db.prepare('DELETE FROM ct_opinion_votes WHERE opinion_id = ?')...
await db.prepare('DELETE FROM ct_opinions WHERE id = ?')...
```

為什麼不倚賴 cascade？因為 SQLite 的外鍵強制執行**預設是關閉的**，要靠連線時的設定打開。Cloudflare D1 有沒有打開、以後會不會變，不是我們能控制的。**能自己保證的事，不要賭別人的預設值。**

---

## 三、API 設計

### 3-1 為什麼投票狀態要獨立一支端點

原本最直覺的做法是把票數加進既有的 `GET /api/issues/:id/opinions`。我沒有這樣做，理由有三：

1. **那支端點的結果會進 SSR。** 伺服器算好 HTML 送給瀏覽器（server-side rendering），但伺服器算的時候不知道你是誰——session 要等瀏覽器帶 cookie 來才知道。把因人而異的資料混進去，伺服器算出來的畫面和瀏覽器接手後算出來的畫面會不一樣，Vue 會警告 **hydration mismatch**（水合不匹配），畫面可能閃爍或壞掉。
2. **那支端點可以被快取。** 公開資料快取起來大家共用很划算；一旦混入「你的票」，就得為每個人各存一份，或乾脆關掉快取。
3. **既有契約不動最安全。** AGENTS.md 的不變量 5 規定既有 API 只能擴充不能改語意。完全不碰它，風險是零。

所以投票狀態走 `GET /api/issues/:id/opinion-votes`，需要登入，而且明確標示不可快取：

```
Cache-Control: private, no-store
Vary: Cookie
```

`Vary: Cookie` 的意思是「這個回應的內容會隨 Cookie 而不同」，提醒中間的快取伺服器不要把 A 的回應拿給 B。

### 3-2 401 和 403 的差別

這兩個很多人搞混：

- **401 Unauthorized**：我不知道你是誰（沒登入）。前端該做的事是**引導登入**。
- **403 Forbidden**：我知道你是誰，但你不能做這件事。前端該做的事是**說明原因**，引導再登入只會繞圈。

這個功能裡：

| 情況 | 回應 |
| --- | --- |
| 沒登入 | 401 |
| 帳號被停權 | 403 |
| 你是提議者，想投自己的意見 | 403 |
| 意見已被判違規 | 403 |
| 票值不是 1／0／-1 | 400（Bad Request，你的請求本身有問題） |
| 意見不存在 | 404 |

### 3-3 守門一定要在伺服器端

前端把按鈕藏起來**不是安全機制**。任何人打開瀏覽器的開發者工具，都能直接對 API 送請求，繞過你所有的畫面邏輯。

所以：

- 前端隱藏提議者的投票鈕 → **體驗**
- 伺服器檢查 `author_id === user.id` 回 403 → **防線**

兩者都要有，但只有後者算數。同理，「投票者是誰」一律從 session 取得，**絕不接受請求裡自己報的 id**——不然任何人都能冒名投票。我特地寫了一個測試守住這件事：

```ts
it('投票者一律取自 session，不吃請求裡自報的 id', async () => {
  await vote({ value: 1, voter_id: 'somebody-else' })
  expect(upsertOpinionVote).toHaveBeenCalledWith({}, 5, 'voter-1', 1)
  //                                                   ↑ session 的 id，不是請求裡的
})
```

### 3-4 一個測試時遇到的意外：DELETE 回 403

我用 curl 測試時發現：

```
POST   /api/opinions/1/vote → 401   （符合預期，沒登入）
DELETE /api/opinions/1/vote → 403   （？）
```

查下去發現是 `src/api/csrf.ts` 的跨站寫入防護擋的，而且**它是對的**。

**CSRF（跨站請求偽造）**是什麼：假設你登入了某銀行網站，接著在另一個分頁打開惡意網站。那個網站可以偷偷發一個請求到銀行，而瀏覽器**會自動帶上你的 cookie**——伺服器看到有效 cookie，就以為是你本人操作。

防禦方式是檢查請求是不是從本站頁面發出的。我的 curl 沒有帶 `Origin` 標頭，看起來就像可疑請求，所以被擋。加上 `Origin` 後就正常回 401 了。真實瀏覽器發同源請求時會自動帶 `Origin`，所以使用者不受影響。

**教訓：看到非預期的結果，先查清楚是 bug 還是保護機制，不要急著改掉。**

---

## 四、「投票前不看分布」是怎麼回事

### 4-1 為什麼要這樣設計

我原本在 issue 裡建議「直接顯示票數」，Bestian 推翻了，指定「投票前不顯示分布，投票後可以改投」。他是對的，理由是**從眾效應（bandwagon effect）**：

> 看到「87 人同意、3 人不同意」，你很難不被影響。你的票就不再是你自己的判斷，而是對群體壓力的反應。

而且這個效應會滾雪球：早期幾票的隨機偏差會被後面的人放大，最後的數字看起來很有代表性，其實只是反映了誰先投。

pol.is 也是這樣設計的。

### 4-2 怎麼實作

關鍵：**隱藏要在伺服器端做**。

```ts
const revealed = isAuthor || myVote !== null
return {
  ...,
  tally: revealed ? { agree, disagree, pass } : null,
  //                                            ↑ 沒表態就連數字都不送出去
}
```

如果只在前端寫 `v-if`，資料還是會被送到瀏覽器，打開開發者工具的 Network 分頁就看得到。**藏在前端等於沒藏。**

### 4-3 但資料還是要公開

Bestian 的設計裡有一個很重要的平衡：不顯示分布**不是保密**，而是**不要在你判斷的當下影響你**。所以任何人都能下載 CSV 原始資料自己算。

這種「讓正確的事情變容易、讓可能有害的事情變麻煩一點，但不禁止」的設計思路，在行為設計裡叫做**摩擦（friction）**的運用。

---

## 五、匿名化

### 5-1 問題

CSV 要公開下載，但裡面有「誰投了什麼票」。如果直接輸出 Better Auth 的 `user.id`，等於公開了每個人的投票紀錄。

### 5-2 解法：序號代號

```ts
export function createParticipantIndex() {
  const seen = new Map<string, number>()
  return {
    seqOf(userId) {
      if (!userId) return 0
      if (seen.has(userId)) return seen.get(userId)
      const next = seen.size + 1
      seen.set(userId, next)
      return next
    },
  }
}
```

第一個出現的人是 `p1`，第二個是 `p2`，以此類推。同一個人在同一份匯出裡永遠是同一個號碼（所以還是看得出「同一個人在不同意見上怎麼投」，這是分群分析必需的），但號碼本身不能反推回任何帳號。

**兩份 CSV 必須共用同一套編號**，否則 `comments.csv` 裡的 `author-id 3` 和 `votes.csv` 裡的 `p3` 會是不同人，分析工具就會算錯。所以我在兩支端點都先跑同一套建立順序：先依意見順序編投稿者，再依投票順序編投票者。

### 5-3 `author_id` 連回應都不能進

前端需要知道「你是不是這則意見的提議者」才能決定畫面。最偷懶的做法是把 `author_id` 送給前端讓它自己比對——**不行**，那等於公開每則意見的投稿者帳號 id。

正確做法是伺服器算完再送結論：

```ts
const isAuthor = row.author_id !== null && row.author_id === viewerId
return { opinion_id, my_vote, is_author: isAuthor, tally }
//                            ↑ 只送布林值，author_id 留在伺服器
```

我也寫了測試守住這件事：

```ts
expect(Object.keys(state).sort()).toEqual(['is_author', 'my_vote', 'opinion_id', 'tally'])
expect(JSON.stringify(state)).not.toContain('author-1')
```

---

## 六、兩個真的很容易踩的陷阱

### 6-1 `0` 是 falsy

JavaScript 裡這些值在 `if` 判斷中都算「假」：`false`、`0`、`''`、`null`、`undefined`、`NaN`。

我們的票值裡，**`0` 代表「略過」，是一個合法且有意義的票**。如果寫成：

```js
if (myVote) { 顯示票數 }   // ❌ 投「略過」的人會被當成沒投票
```

投略過的人就會看不到分布、也沒辦法改票。正確寫法是明確和 `null` 比較：

```js
if (myVote !== null) { 顯示票數 }   // ✅
```

我專門寫了一個測試守住它：

```ts
it('投「略過」（0）也算表態——0 是 falsy，這裡最容易寫錯', () => {
  const state = toOpinionVoteState(row({ my_vote: 0 }), 'viewer-1')
  expect(state.my_vote).toBe(0)
  expect(state.tally).not.toBeNull()
})
```

在真實資料庫上也驗過：user-bob 對意見 902 投了 `0`，查詢正確回傳 `0` 而不是「未投票」。

### 6-2 沒有時區的時間字串

SQLite 存的時間長這樣：`2026-01-02 03:04:05`。注意它**沒有說這是哪個時區的時間**。

JavaScript 的 `new Date('2026-01-02 03:04:05')` 遇到這種字串，不同瀏覽器/引擎的解讀不一樣——有的當 UTC，有的當你電腦的當地時區。在台灣（UTC+8）就會整批差 8 小時。

我們的 `CURRENT_TIMESTAMP` 寫的是 UTC，所以要明講：

```ts
const iso = value.replace(' ', 'T')        // 2026-01-02T03:04:05
Date.parse(`${iso}Z`)                      // 結尾的 Z 就是「這是 UTC」
```

測試：

```ts
it('把沒有時區標記的 SQLite DATETIME 當成 UTC', () => {
  expect(sqliteToMs('2026-01-02 03:04:05')).toBe(Date.UTC(2026, 0, 2, 3, 4, 5))
})
```

---

## 七、測試策略

這次新增 52 個測試，分三種類型，各有各的用途：

### 7-1 黃金樣本測試（golden test）

用在 CSV 上。把期望的輸出**一個字元一個字元寫死**在測試裡：

```ts
expect(csv).toBe(
  'participant,statement_id,vote,updated_at\n' +
    'p1,10,1,2026-01-02T03:04:05.000Z\n' +
    ...
)
```

適用時機：**輸出格式是和外部世界的契約**。以後任何人不小心多加一個空格、改了欄位順序，測試立刻失敗。這比寫「應該包含某某字串」嚴格得多。

### 7-2 純函式單元測試

我把最關鍵的邏輯（當然贊成票、隱藏分布、不外洩 author_id）抽成一個不碰資料庫的函式 `toOpinionVoteState()`，就能直接餵假資料測試各種情況。

**這是一個重要的設計技巧**：把「核心規則」和「資料庫存取」分開，核心規則就變得超好測。抽出來之後我一口氣測了 11 種情況，包括 SQL 在沒有任何票時可能回 `null` 這種邊界。

### 7-3 路由測試（用 mock）

測 401／403／404／400 這些守門規則。做法是把資料庫模組整個換成假的：

```ts
vi.mock('../db/queries', () => ({
  getOpinionVoteTarget: vi.fn(async () => ({ id: 5, author_id: 'author-1', abuse_flagged: 0 })),
  upsertOpinionVote: vi.fn(async () => undefined),
  ...
}))
```

這樣就能檢查「提議者投票時，`upsertOpinionVote` **完全沒有被呼叫**」——不只是回應碼對，而是確認真的沒寫進資料庫。

### 7-4 為什麼還要手動端到端測試

上面三種都是在**假資料庫**上跑的，測不到 SQL 本身有沒有寫錯。所以我另外在本機 D1 塞了真實資料，實際打了兩支 CSV 端點、跑了真實的統計查詢，確認：

- 提議者的當然贊成票真的有加進去（`agrees` 是 2 = 1 真實票 + 1 當然票）
- 被 AI 判定違規的意見**確實沒有出現在 CSV 裡**（內容沒有外洩）
- 沒有任何 Better Auth user id 出現在輸出中
- `0`（略過）和「未投票」在真實 SQL 下確實被區分開

**mock 測邏輯，真實資料庫測 SQL。兩種都要。**

---

## 八、SSR 與 hydration

這個專案的頁面是**伺服器先算好 HTML**送給瀏覽器（SSR），瀏覽器收到後 Vue 再「接手」讓它變成可互動的（這個接手過程叫 **hydration**，水合）。

規則：**伺服器算出來的 HTML，必須和瀏覽器接手時算出來的第一版一模一樣**，否則 Vue 會警告不匹配，畫面可能錯亂。

問題來了：伺服器渲染的時候**不知道你是誰**（登入狀態要等前端問 `/api/me`）。所以我的投票元件第一行是：

```vue
<div v-if="authState !== 'loading'" ...>
```

伺服器端 `authState` 恆為 `'loading'` → 整列不輸出。瀏覽器接手時也還是 `'loading'` → 也不輸出，**兩邊一致**。等 `/api/me` 回來之後才長出來。

實測確認：

```
SSR 中「略過」出現次數（投票鈕專屬字樣，應為 0）：0
```

---

## 九、名詞小辭典

| 名詞 | 白話解釋 |
| --- | --- |
| **migration** | 資料庫的改動腳本。每個檔案一個編號依序執行，讓不同環境的資料庫結構保持一致 |
| **主鍵（PRIMARY KEY）** | 一張表裡用來唯一辨識一列的欄位。複合主鍵就是用好幾個欄位合起來當識別 |
| **upsert** | update + insert：有就更新、沒有就新增，一個動作完成 |
| **競態條件（race condition）** | 兩件事同時發生、互相踩到對方，結果取決於誰先誰後 |
| **原子性（atomic）** | 一個動作要嘛完整發生、要嘛完全沒發生，不會做到一半 |
| **SSR** | 伺服器端渲染：HTML 由伺服器先算好 |
| **hydration** | 瀏覽器接手伺服器算好的 HTML，讓它可以互動 |
| **CSRF** | 跨站請求偽造：惡意網站利用你的登入狀態偷偷送請求 |
| **falsy** | 在 if 判斷裡被當成「假」的值，JavaScript 裡包含 `0` |
| **mock** | 測試時用假的替身取代真實依賴（例如用假資料庫） |
| **黃金樣本測試** | 把期望輸出完整寫死在測試裡，逐字元比對 |
| **正規化** | 資料庫設計原則：把資料拆到合適的表，避免重複儲存 |

---

## 十、這次刻意**沒有**做的事

寫程式很容易「順手」多做一點，但每多做一件事就多一份風險，也讓別人 review 變難。這次明確排除：

- **沒有改 AI 綜整 prompt**（裁示 3）。現有 prompt 明文要求「不因出現次數多就提高重要性」，把票數餵進去等於推翻這條原則。
- **沒有改排序**（裁示 2）。照票數排會讓先貼的意見取得滾雪球優勢，也等於鼓勵灌票。
- **沒有動既有的公開 API 形狀**。投票狀態另開端點。
- **沒有回填既有資料**。提議者的票用算的。
- **沒有做匿名投票**。無法保證一人一票。
- **沒有套用遠端 migration**。依 AGENTS.md 不變量 7，那需要使用者明確授權。

---

## 參考資料

- issue #107 與 [Bestian 的裁示](https://github.com/g0v/civic-talk-hono/issues/107#issuecomment-5772029029)
- [mashbean/pocket-polis](https://github.com/mashbean/pocket-polis) — 投票與 CSV 匯出的參考實作
- [pol.is](https://pol.is/) — 三態投票與意見分群的原始設計
- [Sensemaker](https://make.vtaiwan.tw/) — 吃 pol.is 格式 CSV 的 AI 綜整工具
