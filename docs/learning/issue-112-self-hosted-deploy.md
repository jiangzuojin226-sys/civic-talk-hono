# Issue #112 實作筆記：讓貢獻者能部署到自己的 Cloudflare 環境

> 這份文件記錄 2026-09-23 完成 [issue #112](https://github.com/g0v/civic-talk-hono/issues/112)（PR [#115](https://github.com/g0v/civic-talk-hono/pull/115)）的完整過程：問題怎麼被發現、我讀了哪些原始碼、每個決定背後的原理、實際踩到的五個坑，以及這趟學到的工作方法。
>
> 讀者設定：想理解「為什麼這樣做」而不只是「照著打指令」的人。

---

## 目錄

1. [問題是什麼](#1-問題是什麼)
2. [核心原理：設定檔在「建置時」就被烘焙進產物](#2-核心原理設定檔在建置時就被烘焙進產物)
3. [我提的解法，和維護者的解法](#3-我提的解法和維護者的解法)
4. [驗證方法論：不要用「跑跑看有沒有壞」當測試](#4-驗證方法論不要用跑跑看有沒有壞當測試)
5. [實作全流程](#5-實作全流程)
6. [OAuth 登入的原理](#6-oauth-登入的原理)
7. [踩到的五個坑](#7-踩到的五個坑)
8. [Git 與協作方法](#8-git-與協作方法)
9. [這次沒做完的事](#9-這次沒做完的事)

---

## 1. 問題是什麼

Civic Talk 跑在 Cloudflare Workers 上，資料存在 Cloudflare D1。本機開發指令 `vp run dev` 會用一個**本機模擬的 D1**，但這個模擬資料庫是空的——認證用的 `user`、`session` 等資料表只存在遠端。

後果是：**登入相關的功能在本機完全測不了**。你改了登入流程、角色權限、或任何跟部署環境有關的東西，只能送 PR 給維護者，讓他部署到正式站才知道對不對。這對維護者是負擔，對貢獻者是盲飛。

那能不能自己部署一份？理論上可以——Cloudflare 有免費方案，Workers 和 D1 都夠用。問題卡在設定檔。

專案根目錄的 `wrangler.jsonc` 寫死了正式環境的資訊：

```jsonc
{
  "name": "civic-talk",                                    // 正式 Worker 名稱
  "d1_databases": [
    { "binding": "DB", "database_id": "1b5f4077-..." },    // g0v 帳號的資料庫
    { "binding": "DB_AUTH", "database_id": "e26edd14-..." }
  ],
  "services": [{ "binding": "FACT_CHECK_CORE", "service": "fact-check-core" }]
}
```

這個檔案**在版本控制裡**（tracked）。你要部署到自己的帳號，就得改掉這些值——但改了之後：

- 每次 `git pull` 同步上游都會衝突
- 很容易不小心 `git add .` 把個人設定混進 PR
- 別的貢獻者看到你的 PR 會一頭霧水

所以問題可以精確地表述成：**如何在不修改任何 tracked 檔案的前提下，切換部署目標？**

---

## 2. 核心原理：設定檔在「建置時」就被烘焙進產物

這是整個 issue 的技術核心，也是最容易誤解的地方。

### 直覺的想法（錯的）

大多數人的第一反應是：部署的時候換個設定檔就好了嘛。

```bash
npx wrangler deploy -c wrangler.personal.jsonc   # ← 沒有用
```

**這行指令不會生效。**為什麼？

### 實際發生的事

專案用 `@cloudflare/vite-plugin` 把 Vite 和 Cloudflare 接起來。這個 plugin 在**建置階段**就會讀 wrangler 設定，解析完之後寫成一份新的設定檔放進建置產物：

```
vp run build
   │
   ├─ 讀取 wrangler.jsonc
   ├─ 解析（展開預設值、驗證欄位、計算路徑）
   │
   ├─ 產出 dist/civic_talk/wrangler.json      ← 解析後的完整設定，「烘焙」在這裡
   └─ 產出 .wrangler/deploy/config.json       ← 轉向檔
```

那個轉向檔的內容長這樣：

```json
{ "configPath": "../../dist/civic_talk/wrangler.json", "auxiliaryWorkers": [] }
```

之後你執行 `wrangler deploy`，wrangler 會先看到這個轉向檔，然後**改用它指定的設定**：

```
Using redirected Wrangler configuration.
 - Configuration being used: "dist/civic_talk/wrangler.json"      ← 真正用的
 - Original user's configuration: "wrangler.jsonc"                 ← 只是來源
 - Deploy configuration file: ".wrangler/deploy/config.json"       ← 轉向檔
```

**所以 `-c` 沒用，因為部署階段的設定早就在建置階段決定了。**要換環境，必須在 `build` 那一步就換。

### 「烘焙」這個比喻

想像做蛋糕：`wrangler.jsonc` 是食譜，`vp run build` 是烘焙，`dist/civic_talk/wrangler.json` 是烤出來的蛋糕。`wrangler deploy` 是把蛋糕送出去。

蛋糕烤好之後才說「我想換成巧克力口味」——來不及了，要重新烤。這就是為什麼 `-c` 在部署階段無效。

---

## 3. 我提的解法，和維護者的解法

### 我原本的提案

我讀了 plugin 的型別定義檔 `node_modules/@cloudflare/vite-plugin/dist/index.d.mts`，發現有個 `configPath` 選項：

```ts
interface EntryWorkerConfig {
  configPath?: string;
  // ...
}
```

所以提議改 `vite.config.mts` 一行：

```ts
cloudflare({
  remoteBindings: mode === 'remote',
  configPath: process.env['WRANGLER_CONFIG'],   // 未設定時 undefined，行為不變
})
```

### 維護者的回覆

[@bestian 回覆](https://github.com/g0v/civic-talk-hono/issues/112#issuecomment-5785728647)：plugin 已經內建 `CLOUDFLARE_VITE_WRANGLER_CONFIG_PATH` 環境變數，不用改程式碼。

### 我怎麼查證的

不能因為「維護者說的」就直接照做——文件會叫別人照著做，寫錯就是害人。我去翻了實際安裝的 plugin 原始碼（**不是官網文件，是這台電腦上真正跑的那份程式**）：

```bash
grep -rn "CLOUDFLARE_VITE_WRANGLER_CONFIG_PATH" node_modules/@cloudflare/vite-plugin/dist/
```

找到關鍵那一行：

```js
// dist/index.mjs:64785
configPath = getValidatedWranglerConfigPath(
  root,
  pluginConfig.configPath ?? prefixedEnv.CLOUDFLARE_VITE_WRANGLER_CONFIG_PATH
)
```

`??` 是 **nullish coalescing**：左邊是 `null` 或 `undefined` 才取右邊。

- 左邊 `pluginConfig.configPath` = **我想加的那個**
- 右邊 `prefixedEnv.CLOUDFLARE_VITE_WRANGLER_CONFIG_PATH` = **維護者說的那個**

兩者指向**同一個變數**，只是程式碼設定的優先序高於環境變數。維護者是對的：功能早就有了。

### 而且內建的名字更好

再往上四行：

```js
// dist/index.mjs:64747
const prefixedEnv = vite.loadEnv(viteEnv.mode, root, ["CLOUDFLARE_", "WRANGLER_HYPERDRIVE_..."])
```

Vite 的 `loadEnv` **只載入有指定前綴的變數**。我自己取的名字 `WRANGLER_CONFIG` 沒有 `CLOUDFLARE_` 前綴，所以永遠不會被 `.env` 檔載進來，只能靠 shell 環境變數傳。內建那個名字雖然長，但換來的是可以寫進 `.env` 檔。

### 學到的事

> **在自己造輪子之前，先確認輪子是不是已經在箱子裡。**

我讀了型別定義（`.d.mts`）就停下來了，沒有再讀實作（`.mjs`）。型別定義只告訴你「有哪些選項」，實作才告訴你「這些選項還有沒有別的來源」。多讀那 20 行，就能省下一整段不必要的程式碼改動。

---

## 4. 驗證方法論：不要用「跑跑看有沒有壞」當測試

這趟最值得帶走的不是某個指令，而是一套驗證的思路。

### 原則一：用「假資料」先驗證機制

要確認環境變數真的有用，**不需要先有 Cloudflare 帳號**。我先做了一份填假 ID 的設定檔：

```jsonc
{
  "name": "civic-talk-personal",
  "d1_databases": [
    { "binding": "DB", "database_name": "PLACEHOLDER", "database_id": "00000000-0000-0000-0000-000000000000" }
  ]
}
```

然後建置，看產物：

```bash
CLOUDFLARE_VITE_WRANGLER_CONFIG_PATH=./wrangler.personal.jsonc vp run build
# → 輸出目錄變成 dist/civic_talk_personal/（來自 name 欄位）
```

**輸出目錄的名字就是第一個證據**——它來自設定檔的 `name`，如果環境變數沒生效，目錄會是 `dist/civic_talk/`。

這個做法的價值：把「機制對不對」和「我的帳號設定對不對」**拆成兩件事分別驗證**。如果混在一起，出錯時你不知道是哪一層的問題。

### 原則二：要證明「A 生效」，也要證明「不用 A 時維持原狀」

維護者的檢查清單有一項是「預設建置流程不受影響」。這很容易被忽略——只驗證新功能能用，沒驗證舊行為沒被破壞。

```bash
vp run build                                    # 不帶環境變數
cat .wrangler/deploy/config.json
# → {"configPath":"../../dist/civic_talk/wrangler.json"}   ✅ 切回正式
```

這叫**回歸測試**（regression test）：確認新改動沒有意外改變既有行為。

### 原則三：找「會回顯設定值」的唯讀端點

這招在 debug GitHub 登入時救了我。

`wrangler secret put` 輸入時不顯示字元，貼錯了不會有任何提示。要怎麼確認 Client ID 貼對了？

Better Auth 有個端點會**把伺服器手上的 Client ID 吐出來**：

```javascript
POST /api/auth/sign-in/social  { "provider": "github" }
→ { "url": "https://github.com/login/oauth/authorize?client_id=XXXX&redirect_uri=..." }
```

這個請求**不需要真的登入**，卻能讓你直接比對設定值。比起「點按鈕 → 跳轉 → 看到一頁英文錯誤 → 猜哪裡錯」，這種驗證能直接指出是哪個值不對。

> **通用原則：設定類的東西，找一個唯讀、會回顯設定值的端點來驗證，不要用「跑跑看有沒有壞」當測試。**

### 原則四：驗證要能回答「這一項在證明什麼」

部署完我跑了 12 項煙霧測試，每一項都對應一個具體的疑慮：

| 測試 | 在驗證什麼 |
| --- | --- |
| `/` `/about` 回 200 | SSR 渲染 + `ASSETS` 綁定正常 |
| `/api/issues` 有資料 | **D1 綁定正確**，讀得到自己的資料庫 |
| `/api/me` 回 **401 而不是 500** | 登入守門生效，且 `DB_AUTH` 綁定與 auth 表都正常 |
| `/api/admin/stats` 回 401 | 管理端授權生效 |
| `/api/auth/admin/list-users` 回 **404** | AGENTS.md 要求整段封鎖的端點確實被擋 |
| `/index.html` → 301 `/` | 不變量 4「舊網址永不失效」 |

特別注意第三項：**401 和 500 的差別是關鍵訊號**。兩者都代表「沒進去」，但 401 是「系統正常運作並拒絕了你」，500 是「系統壞了」。如果認證資料庫沒建表，這裡會是 500。用「有沒有回 200」當測試就分不出這兩種情況。

---

## 5. 實作全流程

以下是完整步驟，每一步附上「為什麼」。

### 5.1 從 `upstream/main` 開分支

```bash
git fetch upstream
git switch -c feat/issue-112-personal-wrangler-config upstream/main
```

**為什麼不從當前分支開？** 如果從別的工作分支開，新分支會繼承那個分支的所有 commit，送 PR 時就會夾帶無關的改動。從 `upstream/main` 開，起點就是維護者看到的最新狀態，PR 只會顯示你真正改的東西。

**小陷阱**：這個指令會把新分支的上游設成 `upstream/main`（你沒有寫入權的 repo）。推送時要明確指定：

```bash
git push -u origin HEAD   # 推到自己的 fork，不是 git push
```

### 5.2 先擋，再產生

```bash
# ① 先改 .gitignore
echo "wrangler.personal.jsonc" >> .gitignore

# ② 再建立檔案
cp wrangler.jsonc wrangler.personal.jsonc
```

**順序不能反。**如果先產生檔案再改 gitignore，中間那段時間你如果執行 `git add .`，個人設定就進版控了。Git 有個特性：**已經被追蹤的檔案，後來才加進 `.gitignore` 是無效的**——你得先 `git rm --cached` 才能真的移除。先擋再產生就沒這個問題。

驗證擋住了沒：

```bash
git check-ignore -v wrangler.personal.jsonc
# → .gitignore:13:wrangler.personal.jsonc	wrangler.personal.jsonc
#      ↑ 哪個檔  ↑ 第幾行  ↑ 哪條規則命中
```

這個指令回答「這個檔案**為什麼**被忽略」。比起「`git status` 沒看到它就安心」，這是正面證明——沒出現在 `git status` 也可能是別的原因（例如你路徑打錯了）。

### 5.3 個人設定檔要改四處

| 欄位 | 為什麼要改 |
| --- | --- |
| `name` | Worker 名稱在帳號內唯一。沿用 `civic-talk` 在自己帳號沒事，但養成改名習慣可避免日後誤操作 |
| `d1_databases[DB]` | 換成自己帳號的業務資料庫 |
| `d1_databases[DB_AUTH]` | 換成自己帳號的認證資料庫 |
| `services` | **整段刪除**。`fact-check-core` 只存在於 vTaiwan 帳號，Service Binding 只能綁同帳號資源，帶著它會部署失敗 |

代價：刪掉 `services` 後 `POST /api/fact-check` 在自架環境失效。這是可以接受的取捨——那個功能不是每次都要測。

### 5.4 建立 D1 並套用 migration

```bash
npx wrangler d1 create my-civic-talks
npx wrangler d1 create my-civic-auth
```

**業務庫**用專案的 migration：

```bash
npx wrangler d1 migrations apply my-civic-talks --remote --config wrangler.personal.jsonc
```

套完之後**一定要檢查建了什麼表**（AGENTS.md 不變量 1）：

```bash
npx wrangler d1 execute my-civic-talks --remote --config wrangler.personal.jsonc \
  --command "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name"
```

結果應該是 7 張 `ct_` 開頭的業務表，加上 4 張 D1／SQLite 的內部表（`_cf_KV`、`d1_migrations`、`sqlite_sequence`、`sqlite_stat1`）。

**為什麼這條規則存在？** 正式的 `vtaiwan-civic-talks` 資料庫和 vTaiwan 其他專案共用命名空間。如果有人寫了一個沒加前綴的 `CREATE TABLE issues`，就會撞到別的專案的表，造成**不可逆**的資料破壞。前綴是這個共用環境裡唯一的隔離手段。

**認證庫不能用專案的 migration**：

```bash
# 🚫 絕對不要這樣做
npx wrangler d1 migrations apply my-civic-auth --remote --config wrangler.personal.jsonc
```

因為 `migrations/` 裡全是 `ct_*` 業務表的建表語句，套進認證庫只會建錯東西。auth schema 的唯一來源是 vTaiwan-hono：

```bash
gh api repos/g0v/vTaiwan-hono/contents/migrations/auth --jq '.[].download_url' | xargs -n1 curl -sO
npx wrangler d1 execute my-civic-auth --remote --file 20260727_init_better_auth_tables.sql \
  --config wrangler.personal.jsonc
```

正確結果是四張表：`user`、`session`、`account`、`verification`。

### 5.5 建置 → 預檢 → 部署

```bash
CLOUDFLARE_VITE_WRANGLER_CONFIG_PATH=./wrangler.personal.jsonc vp run build

npx wrangler deploy --dry-run   # 預檢，不會真的上傳
npx wrangler deploy
```

`--dry-run` 要看兩件事：

```
Configuration being used: "dist/civic_talk_personal/wrangler.json"   ← 是個人的嗎？

Your Worker has access to the following bindings:
env.DB (my-civic-talks)        D1 Database    ← 是自己的資料庫嗎？
env.DB_AUTH (my-civic-auth)    D1 Database
env.ASSETS                     Assets
```

沒有 `FACT_CHECK_CORE` 是對的——我們在設定檔裡刪掉了。

### 5.6 設定密鑰

```bash
openssl rand -base64 32 | npx wrangler secret put BETTER_AUTH_SECRET --config wrangler.personal.jsonc
echo "https://你的網址" | npx wrangler secret put BETTER_AUTH_URL --config wrangler.personal.jsonc
```

**順序很重要**：`BETTER_AUTH_URL` 必須等於部署後的 origin，所以得先部署拿到網址，才能設這個值。

`openssl rand -base64 32` 產生 32 bytes 的密碼學隨機值再編成 Base64。為什麼要隨機？這個 secret 用來**簽章** session token——它決定了「這個 cookie 是不是我們發出去的」。可以被猜到的話，任何人都能偽造登入狀態。

---

## 6. OAuth 登入的原理

自架環境要能登入，得自己申請一組 OAuth 應用程式。過程中會遇到兩個容易混淆的概念。

### Client ID vs Client Secret

| | Client ID | Client Secret |
| --- | --- | --- |
| 類比 | 帳號名稱 | 密碼 |
| 會出現在哪 | 使用者的網址列（`?client_id=Ov23li…`），**任何人都看得到** | 只在伺服器和 GitHub 之間傳遞，**永不經過瀏覽器** |
| 格式（GitHub） | 20 字元，`Ov23li` 開頭 | 40 字元，全小寫十六進位 |
| 外洩後果 | 沒事 | 別人能冒充你的應用程式換取使用者授權 |

### PKCE：沒辦法藏 secret 的程式怎麼辦？

`wrangler login` 的授權網址裡有這些參數：

```
client_id=54d11594-...              ← wrangler 自己的 Client ID，公開
redirect_uri=http://localhost:8976/oauth/callback
code_challenge=5HisdVLiv4Bvd1...    ← PKCE
code_challenge_method=S256
state=6.p1XpGr5a3cPu~W0oz...
```

`wrangler` 是裝在你電腦上的程式，**任何 secret 都藏不住**（拆開安裝檔就看得到）。所以它改用 **PKCE**（Proof Key for Code Exchange）：

1. wrangler 產生一個隨機字串（verifier），算出它的 SHA-256 雜湊（challenge）
2. 授權請求只送 challenge
3. 之後拿授權碼換 token 時，才出示原始的 verifier
4. 伺服器驗證 `SHA256(verifier) == challenge`

這證明了「發起請求的」和「來換 token 的」是同一個程式——**不需要事先共享任何 secret**。

`state` 參數是另一回事：防 CSRF。回呼時比對 state 值，確保這次回呼真的是這次請求引發的，不是攻擊者偽造的。

**重點**：整個流程你的 Cloudflare 密碼從未交給 wrangler。它拿到的只是一組有限權限的 token，存在 `~/Library/Preferences/.wrangler/config/default.toml`，隨時可以在後台撤銷。

### Civic Talk 為什麼用傳統的 client secret？

因為它跑在 Worker 上，是**伺服器**，藏得住 secret。secret 存在 Cloudflare 的 secret store，不在程式碼裡——程式碼會進 git，secret 不會。

### `user` 和 `account` 為什麼分兩張表？

```
user  (一個「人」)                       account (一個「登入方式」)
  id: MR2bmE51...  ←─────────────────────  userId: MR2bmE51...
  email: xxx@gmail.com                     providerId: github
  role: user                               accountId: 267463891
                                    ┌────  userId: MR2bmE51...
                                    └───   providerId: google
```

一個 `user` 可對應**多個** `account`。這就是 `accountLinking.trustedProviders: ['google', 'github']` 的作用：同一個 email 用 Google 登入時，會在同一個 `user` 底下新增一筆 `account`，而不是建立第二個使用者。

如果不分表，「同一個人用兩種方式登入」就會變成兩個帳號，投稿紀錄、權限、停權狀態全都對不起來。

---

## 7. 踩到的五個坑

### 坑 1：`wrangler d1 create` 會問要不要幫你改設定檔

```
? Would you like Wrangler to add it on your behalf?
```

回答 **yes** 會把 D1 綁定寫進 `wrangler.jsonc`——那是 tracked 檔案，正是這個 issue 要避免的事。**一律回答 no**，自己填進個人設定檔。

（在非互動環境會自動 fallback 成 no，但你手動跑的時候要注意。）

### 坑 2：新 `*.workers.dev` 子網域的憑證要等

第一次部署成功後，連網址會得到：

```
ERR_SSL_SSL/TLS_ALERT_HANDSHAKE_FAILURE
```

這**不是部署失敗**。`wrangler deploy` 已回報成功就代表 Worker 上線了，只是新子網域的 TLS 憑證還沒簽發完，等幾分鐘就好。

判斷方法：同時測一個已知可連的網址（例如 `https://example.com`）。如果那個通、你的不通，就是憑證問題而不是網路問題。

### 坑 3：環境變數只管 Vite 建置

`CLOUDFLARE_VITE_WRANGLER_CONFIG_PATH` **只有 Vite plugin 會讀**。其他 wrangler 指令完全不看它，預設讀根目錄的 `wrangler.jsonc`——也就是正式環境。

```bash
# ⚠️ 這些都必須自己帶 --config
npx wrangler secret put XXX      --config wrangler.personal.jsonc
npx wrangler d1 execute  XXX     --config wrangler.personal.jsonc
npx wrangler tail                --config wrangler.personal.jsonc
```

**原因**：讀設定的是兩個不同的程式。建置時是 Vite plugin 讀，部署／管理時是 wrangler CLI 自己讀。環境變數只被前者認得。

漏掉 `--config` 的後果是寫到正式環境的 secret 或資料庫。

### 坑 4：決定部署目標的是「最後一次建置」

`dist/civic_talk/` 和 `dist/civic_talk_personal/` 會**同時存在**。真正決定 `wrangler deploy` 上傳哪個的是 `.wrangler/deploy/config.json`，而它只反映**最後一次 build**。

所以「上次用個人設定 build、這次直接 deploy」會部署到個人環境，反之亦然。切換環境時務必重新建置，並用 `--dry-run` 確認。

### 坑 5：看不到輸入的地方最容易貼錯

`wrangler secret put` 輸入時不顯示字元（防偷看）。代價是**貼錯不會有任何提示**——貼上、Enter、顯示 Success，看起來一切正常。

這次就發生了：`GITHUB_CLIENT_ID` 被填成了 Client Secret。症狀是登入導向網址裡的 `client_id` 是 40 位十六進位字元，而不是 `Ov23li…` 開頭的 20 字元。

**更糟的是**：我為了診斷把那個網址印出來，等於把 client secret 印進了對話紀錄。處理方式是到 GitHub 刪掉那組 secret、重新產生一組。

學到兩件事：

1. **看不到輸入的操作，事後要主動驗證**（用第 4 節的「回顯端點」）
2. **診斷輸出可能洩漏機密**。印出完整網址之前先想一下裡面有什麼；真的印出來了就當作已外洩，立刻輪替

---

## 8. Git 與協作方法

### 教訓一：先確認 issue 有沒有人在做

這是從**上一個 issue（#107）的失敗**學到的。我們完整實作了投票功能、送了 PR #111，結果維護者回覆：

> 此 pr 和先前的 pr #110 範圍重複，且有 conflict 存在……未來實作前，可以先看一下議題是否已被認領？還有是否存在同範圍之 pr？

issue #107 早就指派給維護者自己，他也已經開了 PR #110。整趟實作白做。

所以這次動工前先跑這兩個檢查：

```bash
gh issue view 112 --json assignees,state          # 有沒有人認領？
gh pr list --state open --json number,title       # 有沒有同範圍的 PR？
```

十秒鐘的成本，換掉一整次實作的浪費。

### 教訓二：PR 裡每一行都要能解釋

跑格式化工具 `vp fmt` 時，它順手改了 `src/db/queries.ts`——那是維護者剛合併的程式碼，本來就沒照專案的格式規則。如果沒注意，這 12 行無關改動就會混進 PR。

```bash
git diff --stat src/db/queries.ts   # 先看它動了什麼
git checkout -- src/db/queries.ts   # 還原
```

**原則：PR 裡的每一行改動，你都要能回答「這為什麼跟這個 issue 有關」。答不出來就還原。**

### 教訓三：送出前做機密掃描

推送之前，把所有個人識別資訊當關鍵字掃一遍要送出的 diff：

```bash
for pat in "<你的資料庫 ID>" "<你的帳號 ID>" "<你的 Client ID>" "<你的帳號名稱>"; do
  git diff upstream/main | grep -c "$pat"
done
```

資料庫 ID、帳號 ID、Client ID、email——這些都不該出現在公開的 PR 裡。文件要用 `<你的資料庫名稱>` 這種佔位符。

### 教訓四：教學文件和程式碼分開放

這份文件在 `docs/issue-112-learning-notes` 分支，**不會進 PR**。理由：維護者審 PR 時想看的是「這次改了什麼」，混進幾百行教學會讓他很難審。

教學是寫給自己的，不是寫給維護者的。

---

## 9. 這次沒做完的事

誠實記錄，避免日後誤以為都驗過了。

| 項目 | 狀態 | 備註 |
| --- | --- | --- |
| Google 登入 | ❌ 未測 | 需另外到 Google Cloud Console 申請一組 OAuth 憑證 |
| 同 email 帳號整合 | ❌ 未測 | 這是 AGENTS.md「移植進度」5-5 標記待實證的項目——**有了自架環境之後，這是可以幫專案補上的一塊實證**：用同一個 email 分別以 Google 和 GitHub 登入，查 `account` 表是否掛在同一個 `userId` 底下 |
| `POST /api/fact-check` | ❌ 無法測 | Service Binding 指向的 `fact-check-core` 只存在於 vTaiwan 帳號 |
| 投稿安全審查 | ❌ 未測 | 需要 `OPEN_ROUTER_API_KEY`；未設定時採 fail-open（放行並記錄錯誤） |

### 可以接著做的練習

1. **申請 Google OAuth 憑證**，補完登入，順便驗證上面那個「同 email 帳號整合」——這會是對專案有實際價值的貢獻
2. **在自架環境測一次完整投稿流程**：建立議題 → 投稿素材 → 投稿意見 → 送出說明頁，確認四種寫入都落下完整的作者快照（這是 AGENTS.md 9-8 待驗的項目）
3. **故意漏掉 `--config` 看會發生什麼**——在自己的環境安全地體驗一次「差點寫到正式環境」的感覺，比背規則有效

---

## 附錄：本次用到的指令速查

```bash
# 環境
npx wrangler login                                          # OAuth 登入
npx wrangler whoami                                         # 確認帳號

# D1
npx wrangler d1 create <名稱>
npx wrangler d1 list
npx wrangler d1 migrations list <庫> --remote --config wrangler.personal.jsonc
npx wrangler d1 migrations apply <庫> --remote --config wrangler.personal.jsonc
npx wrangler d1 execute <庫> --remote --command "SQL" --config wrangler.personal.jsonc
npx wrangler d1 execute <庫> --remote --file x.sql --config wrangler.personal.jsonc

# 建置與部署
CLOUDFLARE_VITE_WRANGLER_CONFIG_PATH=./wrangler.personal.jsonc vp run build
npx wrangler deploy --dry-run
npx wrangler deploy

# 密鑰
npx wrangler secret put <名稱> --config wrangler.personal.jsonc
npx wrangler secret list --config wrangler.personal.jsonc

# Git 安全檢查
git check-ignore -v <檔案>            # 為什麼被忽略
git diff upstream/main --stat          # 相對上游改了什麼
gh issue view <n> --json assignees     # 有沒有人認領
gh pr list --state open                # 有沒有同範圍 PR
```
