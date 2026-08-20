# CLAUDE.md

本檔案為 Claude Code 在此子資料夾工作時的指引。此資料夾**本身是獨立 git 儲存庫**，不受根目錄工作區規則約束（除語言等全域偏好）。

## 這是什麼

IG 內容規劃報告展示網站。以「品牌顧問」角色，為一家虛構的手沖咖啡店（晨光手沖咖啡）產出一個月（4 週、13 篇）的 Instagram 貼文規劃，並用網頁呈現。**單檔前端，無後端、無序號授權、無 AI 串接**——這是一份顧問報告的展示頁，不是給使用者填表輸入的工具。

## 架構

- `index.html` — 頁面結構與樣式（CSS 變數做暖色調主題，支援淺色/深色）；頁面載入後純前端讀取 `data.js` 渲染四週主題卡片、貼文計畫表格、顧問建議清單。
- `data.js` — **唯一的內容資料來源**：`window.PLANNER_DATA = { brand, weeklyThemes, posts, consultantTips }`。透過一般 `<script src>` 載入（賦值給 `window`），不是 `fetch()`，避開 `file://` 下的 CORS 限制（同 `ai-course-hub` 的「資料即 JS 全域變數」寫法）。

### 要套用到別家店／改內容

只需要改 `data.js`，不用動 `index.html`：

- `brand`：品牌名稱、標語、地點、目前階段、經營型態、客群描述、顧問開場白（`consultantIntro`）。
- `weeklyThemes`：4 個物件，`{week, title, goal}`。
- `posts`：貼文陣列，每篇 `{week, day, type, theme, copy, hashtags}`。`type` 目前僅有三種樣式（`圖文`／`短影音`／`限動`），若要新增類型需同步在 `index.html` 的 CSS 補上 `.type-XXX` 的顏色規則（含深色模式）。`copy` 建議維持 100 字以內、口語親切。
- `consultantTips`：可執行建議清單（字串陣列）。

修改 `data.js` 後，若瀏覽器快取住舊版，把 `index.html` 裡 `<script src="data.js?v=...">` 的版本號往上調。

## 指令

無建置步驟。直接開啟 `index.html`（`file://`）或用伺服器託管即可。

預覽伺服器：port `8796`（見 `.claude/launch.json` 的 `coffee-ig-planner` 項目），用 Preview MCP 的 `preview_start` 啟動，勿手動另起伺服器。
