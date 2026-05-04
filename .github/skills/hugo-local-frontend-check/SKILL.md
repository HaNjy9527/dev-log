---
name: hugo-local-frontend-check
description: '以快速且可重複的流程在本地啟動並驗證這個 Hugo 網誌。當你修改文章、模板、CSS、JavaScript 或 Hugo layouts，且需要預覽前端變更、加快本地啟動與在每次修改後驗證實際頁面時使用。'
argument-hint: '你要在本地驗證哪個前端變更或頁面？'
user-invocable: true
---

# Hugo 本地前端驗證

## 這個 Skill 會產生什麼
- 一套適用於這個 Hugo 專案的快速本地啟動路徑
- 一套每次修改後都能重複使用的前端預覽流程
- 一次針對實際渲染頁面的短驗證，而不只是看 build 結果

## 何時使用
- 你修改了 `content/posts/` 下的 Markdown 內容
- 你編輯了 `assets/ananke/css/custom.css` 內的 CSS
- 你修改了 `layouts/` 下的 Hugo template 或 partial
- 你改了 theme override，且需要驗證實際渲染頁面
- 你想避免本地反覆冷啟動，讓前端迭代更快

## Repo 專屬目標
- 一律從 `my-blog/` 啟動 Hugo，不要從 workspace root 啟動
- 優先使用 workspace task `Hugo: Serve` 與 `Hugo: Build`，不要臨時拼 shell 指令
- 優先維持一個長時間存活的本地預覽 session，而不是每次都完整重啟
- 每次有實質前端變更後，都要在本地瀏覽器驗證頁面
- 預設驗證實際被修改的頁面，而不是泛用首頁

## 流程
1. 先從正確的專案根目錄開始。
   使用 `c:\Users\car88080303\source\repos\MiguelProjectHub\my-blog`。
   如果 Hugo 回報找不到 config file，幾乎可以確定你目前在錯的目錄。
2. 選擇最快的本地預覽路徑。
   優先使用 workspace task `Hugo: Serve`。
   如果 task runner 不可用，再改用：
   ```powershell
   Push-Location "c:\Users\car88080303\source\repos\MiguelProjectHub\my-blog"; hugo server -D --bind 127.0.0.1 --port 1313
   ```
   在編輯期間保持這個 server 持續運作，讓 Hugo live reload 處理大多數變更，避免反覆重啟。
3. 打開這次變更真正影響到的頁面。
   如果是文章專屬變更，不要只停在首頁。
   預設應直接打開被修改的頁面，例如 `/posts/.../` 下的單篇文章。
4. 驗證實際渲染的前端結果，而不是只看原始碼。
   在瀏覽器檢查頁面，確認可見行為符合預期。
   若是互動型變更，要實際驗證互動，例如捲動、點擊、hover 狀態或 responsive 版型。
5. 如果變更涉及結構或腳本，做一次聚焦的瀏覽器驗證。
   例如：
   - 確認目標元素確實存在於 DOM 中
   - 確認預期 class 會在 scroll 或 click 後切換
   - 確認頁面捲動位置或可見狀態有如預期改變
6. 同時檢查桌機與手機寬度下的 responsive 行為。
   手機版驗證是必做，不是可選。
   要確認固定或浮動 UI 不會蓋住內容或頁尾操作區。
7. 如果本地預覽行為正確，再跑一次最終 Hugo build。
   優先使用 workspace task `Hugo: Build`。
   如果 task runner 不可用，再使用：
   ```powershell
   Push-Location "c:\Users\car88080303\source\repos\MiguelProjectHub\my-blog"; hugo
   ```
   這是提交前最後一道防線，用來抓 template 或 asset 問題。
8. 如果瀏覽器驗證與 Hugo build 都通過，就算完成。

## 決策點

### 啟動目錄錯誤 vs 真正的建置失敗
- 如果 Hugo 說找不到 `hugo.toml` 或 config directory，先修正工作目錄。
- 如果 Hugo 能正常啟動但頁面不對，問題就在前端變更本身。

### Live Preview vs Full Build
- 如果你還在迭代 UI，就持續讓 `Hugo: Serve` 保持運作並用本地預覽。
- 如果你認為變更已經完成，就跑一次 `Hugo: Build` 確認最終輸出。

### 頁面範圍
- 如果變更只影響某篇文章，就直接驗證那篇文章頁。
- 如果變更影響共用版型或導覽，至少驗證首頁加一篇文章頁。

### 內容 vs 呈現
- 如果渲染後的 HTML 結構不對，就檢查 Markdown 或 Hugo template。
- 如果結構正確但畫面不對，就檢查 CSS 或 JavaScript 行為。

## 品質檢查
- Hugo 本地 server 能從 `my-blog/` 正常啟動，且沒有 config error
- 在可用情況下，優先使用 workspace task，保持目錄處理一致
- 被影響的前端頁面能在本地正常打開，且已反映最新變更
- 真正的使用者可見行為已在瀏覽器中被檢查
- 有互動的 UI 變更已實際透過 scroll 或 click 驗證
- 被影響頁面已在桌機與手機寬度下檢查
- 結束前最後一次 `hugo` build 成功

## Repo 專屬備註
- Workspace task 定義在 `.vscode/tasks.json`
- 優先使用的 task 名稱是 `Hugo: Serve` 與 `Hugo: Build`
- 若 task 無法執行，才退回直接使用 shell 指令

## 完成條件
- 本地預覽已在受影響頁面上完成驗證
- 前端行為符合預期變更
- 最終靜態 build 乾淨通過

## 範例 Prompt
- `/hugo-local-frontend-check 在本地驗證這篇文章的 CSS 微調`
- `/hugo-local-frontend-check 快速啟動本地預覽並驗證這次模板變更`
- `/hugo-local-frontend-check 檢查我改完回頂按鈕後的文章頁`