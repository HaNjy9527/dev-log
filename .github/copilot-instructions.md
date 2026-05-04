# Copilot Instructions

## Skill Precheck（Plan 前置）
- 在任何需要建立 plan、todo 或多步驟實作的任務開始前，先檢查目前可用 skills。
- 若任務內容符合某個 skill 的描述或觸發情境，必須先使用 `read_file` 讀取該 `SKILL.md`，再進行規劃與實作。
- 若沒有符合的 skill，才可直接進入規劃與實作。

## 全域執行要求（每次任務）
- 若任務包含程式碼或設定修改，完成後至少執行一次最小必要驗證。
- 最終回覆需包含：驗證結果、1 條 Conventional Commit 建議（僅建議，不自動 commit，內容盡量用正體中文）。

## Hugo / 前端變更驗證規則
- 若修改 `my-blog` 專案中的前端相關內容，包含 `content/posts/`、`layouts/`、`assets/`、`static/`、`hugo.toml` 或其他會影響頁面呈現的檔案，結束前必須先做本地預覽驗證，不可只停在 diff 或 build 成功。
- 本地 Hugo 指令必須從 `c:\Users\car88080303\source\repos\MiguelProjectHub\my-blog` 執行；若在 workspace root 執行導致找不到設定檔，需先修正工作目錄再繼續。
- 前端驗證優先使用現有 skill：若需求符合，先讀取 `.github/skills/hugo-local-frontend-check/SKILL.md` 再實作。
- 驗證時預設打開實際被修改的頁面；若為共用版型或導覽變更，至少檢查首頁與一篇文章頁。
- 若變更涉及互動行為、浮動元件、版型、樣式或腳本，需在本地頁面實際驗證互動結果；必要時補桌機與手機寬度檢查。
- 完成本地預覽驗證後，仍需再執行一次 `hugo` 作為最終靜態建置驗證；若無法執行，需明確說明原因與補驗證命令。

## 最小必要驗證
- `.NET` 變更：執行 `dotnet build Car1.Finance.ReconPortal/Car1.Finance.ReconPortal.csproj`。
- 若無法驗證，需明確註記原因，並給出可執行的補驗證命令。

## 文件產出規範
- 產生專案文件時，預設使用 Markdown（`.md`）。
- 文件預設放在 `doc/` 目錄（若使用者指定路徑則依使用者要求）。
- 文件內容至少包含：標題、建立時間（台灣時間 ISO 8601 +08:00）、章節大綱。

## SQL 寫入安全規則
- 新增或修改任何 SQL 寫入操作（INSERT / UPDATE / DELETE / MERGE / TRUNCATE / BulkCopy）前，**必須先讀取** `doc/底層計畫需求/資料庫寫入範圍清單.md`，確認操作不超出允許範圍。
- 若需寫入清單以外的資料表或欄位，須先更新該文件並經使用者確認後才可實作。

## C#/.NET 變更準則（精簡）
- 優先採最小改動，避免無關重構。
- 命名與縮排遵循既有程式碼風格。
- 非同步方法使用 `Async` 後綴，避免 `async void`（事件處理除外）。
