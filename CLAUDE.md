# CLAUDE.md

本檔案為 Claude Code 在此子資料夾工作時的指引。此資料夾**本身是獨立 git 儲存庫**，不受根目錄工作區規則約束（除語言等全域偏好）。

## 這是什麼

社群內容規劃報告產生器。以「品牌顧問」角色，依使用者在頁面上填的品牌變數，產生一個月（4 週、至少 12 篇）的 Instagram 貼文規劃。**單檔前端，無後端、無序號授權**。預設示範品牌為虛構的「晨光手沖咖啡」。

兩種產生方式（同一顆「產生我的四週計畫」按鈕，依是否填 AI 金鑰自動切換）：

- **規則式套版（免 AI）**：把 `data.js` 範本裡 13 篇貼文的 `{{BRAND}}` token 換成使用者填的品牌名稱，週主題／貼文結構／建議完全不變。沒填 AI 金鑰時自動走這條路，AI 呼叫失敗時也會自動退回這條路（不會讓頁面沒有內容）。
- **AI 生成（選用，BYOK）**：使用者自備 Claude／OpenAI／Gemini／OpenRouter 的 API 金鑰，把品牌變數整包送給 AI，請它依原始規格（4 週、每週≥3篇、每篇≤100字、3-5個hashtag、親切口語）重新生成一份完全客製的週主題／貼文／顧問建議／顧問筆記。

## 架構

- `index.html` — 表單（品牌變數＋選用的 AI 補充描述）＋ AI 金鑰設定＋渲染區（頁首／顧問筆記／4 週卡片／貼文表格／建議清單）。單一 IIFE `<script>`。
- `data.js` — `window.PLANNER_TEMPLATE = { defaultBrand, weeklyThemes, posts, consultantTips }`。`posts[].copy` 與 `posts[].hashtags[]` 裡用 `{{BRAND}}` 標記品牌名稱出現的位置，只有規則式套版會替換這個 token；AI 生成時完全不用這份範本文字，由 AI 直接產生最終內容。

### 核心函式（`index.html` 內）

- `applyTemplate(template, brand)` — 規則式套版：`{{BRAND}}` token 替換＋`buildConsultantIntro(brand)` 組顧問筆記。
- `renderAll(brand, content)` — 唯一的渲染入口，載入時與每次「產生」/「重設」後都會呼叫。
- `AI_PROVIDERS` / `callLLM(provider, model, apiKey, prompt, onRetry)` / `parseJsonObject()` — 直接比照 `SocialPost/index.html` 與 `product-title-generator/index.html` 已驗證過的 BYOK 實作搬過來（Claude 需要 `anthropic-dangerous-direct-browser-access` header 才能瀏覽器直連；OpenAI/Gemini/OpenRouter 無此限制；429/500/503/529 自動重試 3 次）。修改任一邊的 AI 引擎邏輯時，考慮是否也要同步另外兩邊。
- `buildPrompt(brand, extra)` — 组 AI prompt，要求回傳純 JSON（`consultantIntro`/`weeklyThemes`/`posts`/`consultantTips`）。
- `validateAiContent(obj)` — AI 回傳 JSON 的安全網覆核（週數／每週篇數／`type` 合法值／hashtag 數量／文案長度），不完全信任 prompt 指示；缺必要欄位就 `throw`，由呼叫端 catch 後退回規則式套版並顯示錯誤原因。

### localStorage

- `coffeeIgPlannerBrand` — 目前的品牌變數表單值。
- `coffeeIgPlannerContent` — 目前渲染中的內容（規則式或 AI 生成的結果都會存，reload 後還原，不會回到預設範例）。
- `coffeeIgPlannerApiConfig` — `{provider, model, apiKey}`，金鑰只存本機瀏覽器，不經任何後端。
- `coffeeIgPlannerMarquee` — 跑馬燈公告快取（見下方「頂部跑馬燈」）。
- `coffeeIgPlannerSavedPlans` — 「已儲存的計畫」多筆具名清單，見下方「儲存與下載」。

「重設為範例」按鈕會清掉前三個 key 並還原成 `defaultBrand`。

## 5 組快速範例

`data.js` 的 `window.PLANNER_TEMPLATE.presets`（5 組）驅動表單上方的 `.example-btn` 藥丸按鈕（`initExamples()`）。**刻意全部維持在咖啡／飲品店這個大類**（不同經營型態與客群變化），因為規則式套版只替換 `{{BRAND}}` token、不改寫文案情境——如果範例跨到完全不同產業（例如瑜伽教室），沒填 AI 金鑰時套出來的貼文會文不對題。真正想做跨產業的內容，本來就該搭配 AI 生成（表單本身沒有產業限制）。每組 preset 除了 8 個品牌欄位，還有一個 `extra` 欄位（只在 AI 生成時使用，寫入 `#f-extra`）。

## 儲存與下載

- **已儲存的計畫**（`.saved-plans` 區塊，`initSavedPlans()`）：多筆具名清單，存 `coffeeIgPlannerSavedPlans`（`{id,name,savedAt,brand,content,extra}[]`）。按「💾 儲存目前計畫」存一筆（同名會詢問覆蓋）；清單用 `#savedList` 單一事件委派（`data-action="load|download|rename|delete"`）處理，DOM 一律用 `createElement`/`textContent` 組出（不用字串拼接 `innerHTML`，避免使用者輸入的名稱或 AI 生成內容裡若含 `<`/`>` 造成注入）。跟現有的單筆自動存檔（`coffeeIgPlannerBrand`/`Content`）是兩套獨立機制，互不影響。
- **下載目前計畫**（`#downloadBtn` → `onDownloadClick()`）：`buildTextReport(brand, content)` 把品牌資訊／顧問筆記／四週主題／完整貼文列表／顧問建議組成一份純文字報告，`downloadText()` 用 `Blob` + 隱藏 `<a download>` 觸發瀏覽器下載（不經任何伺服器）。清單裡每一筆的「下載」按鈕呼叫同一組函式，直接用該筆存的 `brand`/`content`，不需要先載入才能下載。檔名經 `sanitizeFilename()` 去除 `\/:*?"<>|`。

## 頂部跑馬燈

比照 `SocialPost/index.html` 已驗證的共用實作：`MARQUEE_CHECK_URL` 是工作區多個工具共用的同一顆 Google Apps Script 端點，頁面載入時 POST 空 `serial`（本工具無序號機制，`doPost` 不論序號有效與否都會附上 `marquee` 陣列），只取回傳的 `marquee` 陣列（字串陣列）。先讀 `localStorage` 快取立即顯示，背景每 20 分鐘重抓一次；抓取失敗靜默忽略。獨立 `<script>` IIFE，跟主程式邏輯互不相依。**本頁沒有 `.topbar`／sticky header**，所以只需要 `body.has-marquee{padding-top:30px}`，不用像有 sticky topbar 的姊妹工具那樣額外調整 `.topbar{top:30px}`。改跑馬燈內容直接編輯共用 Google Sheet，不需要重新部署 Apps Script。

**2026-08-20 更新（`Code.gs` 未改動、不需重新部署）**：`render()` 新增 `lastKey`（`JSON.stringify(items)`）比對，內容沒變就不重繪，CSS animation 不再被重置歸零重跑；新增 `appendParsedText()`／`buildTrackContent()` 支援 `[文字](https://...)` 連結語法（`createTextNode` 組 DOM，避免 XSS），資料格式仍是純字串陣列，向下相容。已 commit＋push（GitHub Pages 自動重新部署）。

## 操作手冊（manual.html）

自成一頁、內嵌 `<style>`（沿用本頁的淺色暖色調變數，非 `Prompt/manual.html` 的深色主題）。內容涵蓋操作步驟／5 組快速範例說明／加入主畫面說明／隱私說明／使用警語／創作者資料／授權限制。**創作者資料區塊逐字比照** `Prompt/manual.html`／`SocialPost/manual.html`／`sbir-generator/manual.html`／`icap-generator/manual.html`／`phoenix-loan-generator/manual.html`，更新其中一邊時同步其餘各邊。`index.html` footer 的 `.footer-meta` 右側有連結 `manual.html`。

## 加入主畫面（PWA）

`manifest.json`＋`service-worker.js`（network-first＋同源快取備援，`fetch(request,{cache:'reload'})` 這個細節必須保留，否則 GitHub Pages 的 HTTP 快取會讓 network-first 失效）＋`icons/`（PIL＋`C:\Windows\Fonts\msjhbd.ttc` 產生，暖棕色底＋米白色單字「顧」，192／512／maskable-512／apple-touch-icon 四種尺寸；產生腳本未進 repo，比照 `SocialPost` 慣例）。安裝按鈕 `#installBtn`（footer）＋獨立 `#toast` 元素＋逐字沿用 `SocialPost` 已修好 bug 的安裝腳本（自帶 `notify()`，不依賴外部 `showToast()`，避免重蹈其他姊妹工具「按鈕沒反應」的舊坑）。**已用瀏覽器實測確認**：manifest／service worker 皆正確註冊為 active，Chrome 判定頁面符合安裝條件並觸發真實 `beforeinstallprompt`（測試時用合成 click 觸發 `.prompt()` 因缺少真實使用者手勢而報 `NotAllowedError`，這是測試方法本身的限制，不是功能缺陷——真人點擊時會正常運作）。

## 響應式（RWD）

版面本來就用 `repeat(auto-fit,minmax(...))` 的 grid（`.form-grid`／`.api-grid`／`.week-grid`）＋`flex-wrap:wrap`（`.brand-meta`／`.example-row`／`.form-actions`／`.footer-meta`）＋貼文表格獨立 `overflow-x:auto` 容器，天生對窄螢幕友善。**已用 iframe 模擬視窗寬度的方式實測驗證**（`resize_window` 在本機環境對此分頁無效，改用建立指定寬度的 `<iframe src="...">` 量測 `contentDocument.documentElement.scrollWidth` 判斷是否溢出）：320px／375px／768px 下 `index.html` 與 `manual.html` 皆無水平溢出，表格在自己的容器內橫向捲動、不撐開頁面，grid 於 320px 收成單欄、768px 展開多欄。

## 部署

已推公開 GitHub repo：<https://github.com/M255525/coffee-ig-planner>，已用 `.github/workflows/deploy-pages.yml`（Actions 部署模式，比照 `workspace-git-repos` 記載的「不要用 legacy branch-source」慣例）啟用 GitHub Pages：<https://m255525.github.io/coffee-ig-planner/>。頁尾有訪客次數計數器（`visitor-badge.laobi.icu` 的 SVG badge，`page_id=m255525.coffeeigplanner`，免金鑰免後端，比照 `SocialPost`／`mrvideo_s` 既有慣例）。

## 指令

無建置步驟。直接開啟 `index.html`（`file://`）或用伺服器託管即可。

預覽伺服器：port `8796`（見 `.claude/launch.json` 的 `coffee-ig-planner` 項目），用 Preview MCP 的 `preview_start` 啟動；若該 MCP 在當次工作階段不可用，退回 `python -m http.server 8796 --directory 行銷內容工具/coffee-ig-planner` 暫起、測完關閉。

驗證 AI 路徑不需要真實金鑰：可在瀏覽器 console 攔截 `window.fetch` 回傳假的 provider 回應格式（例如 OpenAI 的 `{choices:[{message:{content: JSON.stringify(...)}}]}`），確認 `callLLM → parseJsonObject → validateAiContent → renderAll` 整條管線正確，測完記得還原 `window.fetch`。
