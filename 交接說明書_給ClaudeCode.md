# 建置說明書：Notion 合約＋報價單自動生成系統

> **這份文件是寫給 Claude Code（或其他 AI 助手）看的。**
> 人類使用者：把這份文件貼給你的 Claude Code，說「照這份說明書幫我建置」，然後準備好第二節列出的東西。
> 系統已在原環境完整驗證過，本文含所有已知的坑，照做即可，不需重新摸索。

---

## 〇、系統概觀

**效果**：在 Notion 資料庫填客戶資料 → 按按鈕 → 15~30 秒後合約／報價單 PDF 自動出現在同一列。

**架構**（合約與報價單各一組，完全獨立）：

```
Notion 按鈕（動作＝把勾選欄位打勾）
→ [偵察兵 workflow] Schedule 每 15 秒
   → 查資料庫（filter: 勾選=true）
   → Code 防重複（staticData，10 秒窗口）
   → 先把勾收掉（防重複認領）
   → POST 叫醒主力 webhook
→ [主力 workflow] Webhook 收 {data:{id:頁面id}}
   → GET Notion 頁面（拿全部欄位＋檔案的臨時簽名網址）
   → Code 組參數網址（含防偽驗證碼），指向 GitHub Pages 上的 HTML 模板
   → POST Gotenberg /forms/chromium/convert/url 轉 PDF（回傳 binary）
   → Notion File Upload API 三步上傳 PDF
   → PATCH 頁面：檔案欄掛 PDF、取消勾選、狀態=已產生
```

**為什麼這樣設計**（省得重新踩坑）：
- n8n 的 Notion 輪詢觸發器**硬性最少 1 分鐘**，要秒級反應必須用 Schedule Trigger（允許秒級）自己查，所以拆成偵察兵＋主力兩條
- Notion 按鈕的「Send webhook」動作在原環境測試**送不出去**（原因未明），所以按鈕動作用「編輯屬性→打勾」，由偵察兵撿
- Notion 檔案網址一小時過期，所以一切「當下抓當下用」，PDF 生成後永久保存在 Notion
- 模板是純靜態 HTML（URL 參數帶資料），人可以開網頁預覽，Gotenberg 也用同一份轉 PDF，一份模板兩用

---

## 一、素材（全部公開，直接抓）

Repo：`https://github.com/smallwins33/contract-generator`（raw 檔案可直接 curl）

| 檔案 | 用途 |
|---|---|
| `contract.html` | Charlie 版合約模板 |
| `contract_yoyo.html` | Yoyo 版合約模板 |
| `quote.html` | 費用報價單模板 |
| `n8n_workflow_contract.json` | 合約主力 workflow |
| `n8n_workflow_scout.json` | 合約偵察兵 |
| `n8n_workflow_quote.json` | 報價主力 |
| `n8n_workflow_quote_scout.json` | 報價偵察兵 |
| `notion_fields_n8n.txt` | 合約資料庫欄位清單 |
| `操作守則.md` | 給操作人員的日常使用說明 |

---

## 二、人類要先準備的東西（Claude Code 代辦不了）

1. **n8n 實例**＋API key（Settings → n8n API → Create key）。推薦 Zeabur/Docker 自架
2. **Gotenberg**：正式環境必須自架（n8n 同一個 Zeabur 專案／docker-compose 加開 `gotenberg/gotenberg` 容器）。測試期可暫用 `https://demo.gotenberg.dev`（限流、個資會經第三方，只能餵假資料）
3. **Notion integrations**（notion.so/profile/integrations 建立，拿 `ntn_` token）：
   - 合約一把：合約資料庫開「連接」給它
   - 報價一把：報價資料庫＋**所有被關聯的來源資料庫**（月子中心、房型、醫生、醫院、費用設定）都要各開一次「連接」，否則讀不到關聯頁名稱、rollup 也建不了
4. **GitHub 帳號**：開一個公開 repo 放模板，開 GitHub Pages（模板網址要讓 Gotenberg 開得到）
5. 把以上的 token／API key／網址交給 Claude Code

## 三、Notion 資料庫欄位契約（名稱必須一字不差）

**欄位名稱＝程式介面。改名必須同步改 workflow 的 Code 節點。**

### 合約資料庫（欄位詳表見 notion_fields_n8n.txt）

- 讀取：`中文姓名`(title)、`護照英文姓名`、`西元生日`(date)、`護照號碼`、`通訊地址`、`電話`(phone)、`Email`(email)、`陪客-大`、`陪客-小`(number)、`緊急聯絡人`、`緊急聯絡人電話`、`過敏原`(text)、`待產入住日期`、`月子入住日期`(date，**支援區間**)、`待產天數`、`月子天數`(number 或 formula 皆可)、`待產房型`、`月子房型`(select)、`待產每日房費`、`月子每日房費`、`定金`(number)、`月子中心`(select: Charlie/Yoyo/Johnny)、`證件照`(files)
- 寫入：`合約 PDF`(files)、`產生合約`(checkbox)、`合約狀態`(select: 待產生/已產生/產生失敗)

### 報價資料庫（金額全部 Notion 公式算好，n8n 只讀結果）

- 讀取：`客戶姓名`(title)、`待產期間`、`月子期間`(date 區間)、`待產天數`、`月子天數`、`醫生費用`、`醫院費用`、`小費`、`陪客費`、`💰總計美元`(formula)、`單價待產/日`、`單價月子/日`、`設定證件代辦`、`設定政府規費`、`設定匯率`(rollup)、`其他費用`、`陪客大人人數`、`陪客小孩人數`(number)、`生產方式`(select)、`⚠️檢查`(formula string)、`名稱月子中心`、`名稱待產房型`、`名稱月子房型`、`名稱醫生`、`名稱醫院`(rollup→關聯標題，function=show_original)
- 寫入：`報價單 PDF`(files)、`產生報價單`(checkbox)、`報價狀態`(select)

---

## 四、建置步驟（Claude Code 執行）

1. **部署模板**：從素材 repo 抓三個 html → push 到使用者的公開 repo → 開 GitHub Pages（`gh api repos/{owner}/{repo}/pages -X POST -f "source[branch]=main" -f "source[path]=/"`）→ curl 確認 200
2. **建 n8n 憑證**：`POST /api/v1/credentials`，type `notionApi`，data `{"apiKey":"ntn_..."}`（合約、報價各一個）
3. **匯入四條 workflow**：`POST /api/v1/workflows`，內容用素材的 JSON，匯入前修改：
   - Code 節點裡的 `BASE` → 使用者的 GitHub Pages 網址
   - 偵察兵的 query 網址 → 使用者的資料庫 ID；「叫醒主力」網址 → 使用者的 n8n webhook 網址
   - 「轉成 PDF」節點網址 → 使用者的 Gotenberg
   - 每個節點掛上對應憑證：`"credentials": {"notionApi": {"id": "...", "name": "..."}}`
4. **啟用**：`POST /api/v1/workflows/{id}/activate`，然後 curl 打 webhook 確認回 200
5. **Notion 端按鈕**（人類手動）：資料庫加「按鈕」欄位 → 動作「編輯屬性」→ 把 `產生合約`／`產生報價單` 設為已勾選；把勾選欄位隱藏
6. **端到端測試**（見第六節）

---

## 五、已知的坑（每一條都是實際踩過的）

1. **n8n Code 節點沙箱沒有 `URLSearchParams`** → 組網址一律用 `encodeURIComponent` 手動串（素材 JSON 已如此）
2. **用 API 建立含 Webhook 節點的 workflow，節點必須手動塞 `webhookId`（隨機 UUID）**，否則啟用後 webhook 不會註冊、打了回 404
3. **n8n workflow 停用再啟用可能殘留舊計時器** → 偵察兵同秒重複觸發。改排程類 workflow 時**刪掉重建**，不要停用重啟
4. **模板端不可對 URL 參數做第二次 `decodeURIComponent`**：`URLSearchParams.get()` 已解碼一次，再解會弄壞 Notion 帶簽名的檔案網址（`%2F` 變 `/`，簽名失效、圖片載不出）
5. **證件照 `<img>` 的 `crossorigin` 屬性**：Notion 檔案伺服器不回 CORS 標頭，掛了會整張拒載。模板已做「先試 crossorigin、onerror 退回一般載入」
6. **Notion formula／rollup 欄位讀法**：`prop.formula.number`、`prop.rollup.number`、rollup 標題在 `prop.rollup.array[0].title[].plain_text`。數字欄用容錯讀法 `x?.number ?? x?.formula?.number ?? 0`
7. **關聯來源資料庫沒開「連接」時**：API 完全看不到（404）、rollup 也建不了，錯誤訊息是 `Related collection not found`。解法就是去每個來源資料庫開連接
8. **HTML 轉 PDF 的分頁**：跨列合併格（rowspan）被切頁會欄位錯位 → 把每個分類群組拆成獨立小表格（相同 colgroup 固定欄寬、`page-break-inside: avoid`），視覺上仍像一張表。模板已處理，改表格時遵守此模式
9. **浮水印高度**：鋪到 `doc.scrollHeight` 為止就好，多鋪會在 PDF 尾端多出一頁只有浮水印的空白頁
10. **列印模式 CSS**：`@media print` 要隱藏按鈕／預覽橫幅、歸零內距（頁邊距交給 Gotenberg 參數 0.45in；報價單模板設 0）
11. **Notion File Upload API 三步**：`POST /v1/file_uploads` {mode:single_part, filename} → `POST /v1/file_uploads/{id}/send`（multipart 傳 binary）→ PATCH 頁面 files 掛 `{type:"file_upload", file_upload:{id}}`。Notion-Version 用 `2022-06-28`
12. **防偽驗證碼**：模板和 n8n Code 用同一條算式算 `v` 參數，不帶 v＝預覽模式（紅色警示橫幅），v 錯＝顯示錯誤頁。改參數結構時兩邊要同步
13. **報價單防呆**：`⚠️檢查` 欄含 🚨 就 throw 不生成；另外用獨立算式重算總計、跟 Notion 公式對不上也 throw（防公式被改壞）。注意：被擋下時 Notion 端沒有任何提示，錯誤只在 n8n 執行紀錄
14. **Gotenberg demo 只能測試**：每秒 2 次限流、第三方伺服器。正式必須自架，只改「轉成 PDF」節點的網址一處

---

## 五-b、自架 Gotenberg 實際步驟（正式上線前必做）

Gotenberg 是開源的「HTML 轉 PDF 引擎」（內建無頭 Chrome），官方 Docker 映像檔：`gotenberg/gotenberg:8`。不用任何設定檔，開起來就能用。

### 情境 A：n8n 跑在 Zeabur（原環境的情境）

1. 進 n8n 所在的 Zeabur 專案 → **Add Service** → 選 Docker Image → 填 `gotenberg/gotenberg:8` → 部署（它聽 port 3000）
2. **優先用專案內部網路**：同專案的服務可以用私有網路互連（主機名通常是服務名稱，例如 `gotenberg.zeabur.internal:3000`，依 Zeabur 介面顯示為準）——這樣 Gotenberg 完全不對外公開，最安全
3. 若私有網路不可用，退而求其次綁一個公開網域——但要知道：**Gotenberg 本身沒有密碼保護**，公開網址等於任何人都能用你的機器轉檔（資安風險低、但有被濫用吃資源的帳單風險），建議網域取隨機難猜的名稱
4. 到 n8n 兩條主力 workflow 的「轉成 PDF」節點，把網址從
   `https://demo.gotenberg.dev/forms/chromium/convert/url`
   改成
   `http://（內部主機名）:3000/forms/chromium/convert/url`

### 情境 B：n8n 用 docker-compose 自架

```yaml
services:
  n8n:
    image: n8nio/n8n
    # ...原有設定...
  gotenberg:
    image: gotenberg/gotenberg:8
```

n8n 裡的網址填 `http://gotenberg:3000/forms/chromium/convert/url`（compose 服務名即主機名）。

### 驗證

用 n8n 手動執行一次主力 workflow（或 curl 打 Gotenberg）：

```bash
curl -X POST "http://<主機>:3000/forms/chromium/convert/url" \
  -F url=https://<你的 Pages 網址>/contract.html -F printBackground=true \
  -o test.pdf
# test.pdf 開得起來、file test.pdf 顯示 "PDF document" 即通
```

注意：Gotenberg 要能連外（它得打開 GitHub Pages 的模板網址），自架環境如有防火牆要放行對外流量。

---

## 五-c、GitHub Pages 的限制與隱私強化選項（建置者必讀，向業主說明後選擇）

本系統把模板放在 GitHub Pages，好處是免費、零維護、人和 Gotenberg 都開得到。代價如下，**請向業主（月子中心）說明並讓他選擇等級**：

### 弱點

1. **免費版強制公開 repo**：原始碼可被瀏覽——模板裡含商業資訊（收款銀行帳號、公司統編、代表人、地址、電話、價目、合約條款全文）
2. **模板網頁是公開網址**：知道網址就看得到空白合約；帶參數的正式連結**本身含客戶個資**，連結傳到哪、資料就到哪
3. 帶參數的網址會出現在 GitHub 伺服器紀錄（不對外，但存在）
4. 可用性依賴 GitHub：Pages 故障＝生成停擺
5. ~~搜尋引擎收錄~~ → 已處理：三個模板都有 `<meta name="robots" content="noindex, nofollow">`

### 解法階梯（由簡到繁，依業主隱私要求選）

| 等級 | 做法 | 擋掉什麼 | 剩什麼 |
|---|---|---|---|
| 0（已內建） | 模板 noindex | 搜尋引擎收錄 | 知道網址仍可開 |
| 1 | repo 轉私有＋改用 **Cloudflare Pages** 部署（免費支援私有 repo；或 GitHub Pro $4/月） | 原始碼瀏覽與索引 | 模板網頁本身仍公開 |
| 2 | 頁面前加 **Cloudflare Access**（免費版可用），Gotenberg 請求時在「轉成 PDF」節點加 `extraHttpHeaders` 表單欄位帶 service token | 一般人開網址 | 設定複雜度↑，預覽要登入 |
| 3（最徹底） | **完全內網化**：不放公開網頁。模板改成「資料嵌入模式」（把 URL 參數讀取改為讀 `window.DATA`，n8n 在 Code 節點把資料 JSON 塞進模板字串），改用 Gotenberg 的 `/forms/chromium/convert/html` 路由直接上傳 HTML | 上述全部——零公網暴露，個資不出內網 | 失去「開網址即預覽」功能；模板改版要動 n8n 內的模板來源 |

### 建議

- 一般情況：**等級 0＋1** 即可（源碼不公開、不被搜尋，模板網址難猜且個資僅存在於連結流轉過程）
- 業主對客戶個資有明確高要求（或有法遵考量）：直接做**等級 3**，架構最乾淨——反正正式環境已要求自架 Gotenberg，內網化只是把模板也收進來

---

## 六、驗收清單（全過才算建置完成）

- [ ] 合約：填假資料＋上傳一張圖到證件照 → 按按鈕 → 30 秒內「合約 PDF」出現、勾自動取消、狀態=已產生
- [ ] 打開 PDF：資料正確帶入、金額計算正確、證件照頁有圖＋紅色姓名浮水印、每頁有灰色浮水印、無空白頁、附表無錯位
- [ ] 月子中心切 Yoyo 再按 → 產出 Yoyo 版（抬頭 EVERWYN INC.、頭款 70% 扣定金）
- [ ] 再按一次 → 覆蓋重新生成；10 秒內連按 → 只生成一份
- [ ] 報價單：範例列（費用設定=標準、⚠️檢查=✅）按按鈕 → 報價單 PDF 出現、名稱欄位（中心/房型/醫生/醫院）有帶出、總計與 Notion 一致
- [ ] 防呆：把費用設定清空（⚠️檢查=🚨）再按 → 不生成，n8n 紀錄顯示錯誤原因
- [ ] 竄改測試：把合約網址的金額參數改掉 → 顯示「無法驗證」錯誤頁

---

## 七、原環境參數（對照用）

- 模板：`https://smallwins33.github.io/contract-generator/`
- n8n：`https://n8n-miyavi.zeabur.app`，workflow：合約主力 `yYZfzl4W9lZhjQwd`／合約偵察兵 `9fwL8tlb2R6Wopki`／報價主力 `XFVztTJ9Gh2zqPNU`／報價偵察兵 `JOTODYVzBW465Y1f`
- webhook 路徑：`/webhook/contract-generate`、`/webhook/quote-generate`
- 交接後這些全部換成新環境的值；舊環境的 token 記得撤銷
