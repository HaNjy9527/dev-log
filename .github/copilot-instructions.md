# Copilot Instructions

## Skill Precheck（Plan 前置）
- 在任何需要建立 plan、todo 或多步驟實作的任務開始前，先檢查目前可用 skills。
- 若任務內容符合某個 skill 的描述或觸發情境，必須先使用 `read_file` 讀取該 `SKILL.md`，再進行規劃與實作。
- 若沒有符合的 skill，才可直接進入規劃與實作。

## 全域執行要求（每次任務）
- 若任務包含程式碼或設定修改，完成後至少執行一次最小必要驗證。
- 最終回覆需包含：驗證結果、1 條 Conventional Commit 建議（僅建議，不自動 commit，內容盡量用正體中文）。

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
