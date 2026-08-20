# CLAUDE.md

本檔案為 Claude Code 在此子資料夾工作時的指引。此資料夾**本身是獨立 git 儲存庫**，不受根目錄工作區規則約束（除語言等全域偏好）。

## 這是什麼

IG 內容規劃報告產生器。以「品牌顧問」角色，依使用者在頁面上填的品牌變數，產生一個月（4 週、至少 12 篇）的 Instagram 貼文規劃。**單檔前端，無後端、無序號授權**。預設示範品牌為虛構的「晨光手沖咖啡」。

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

「重設為範例」按鈕會清掉這三個 key 並還原成 `defaultBrand`。

## 指令

無建置步驟。直接開啟 `index.html`（`file://`）或用伺服器託管即可。

預覽伺服器：port `8796`（見 `.claude/launch.json` 的 `coffee-ig-planner` 項目），用 Preview MCP 的 `preview_start` 啟動；若該 MCP 在當次工作階段不可用，退回 `python -m http.server 8796 --directory 行銷內容工具/coffee-ig-planner` 暫起、測完關閉。

驗證 AI 路徑不需要真實金鑰：可在瀏覽器 console 攔截 `window.fetch` 回傳假的 provider 回應格式（例如 OpenAI 的 `{choices:[{message:{content: JSON.stringify(...)}}]}`），確認 `callLLM → parseJsonObject → validateAiContent → renderAll` 整條管線正確，測完記得還原 `window.fetch`。
