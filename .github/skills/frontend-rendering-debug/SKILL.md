---
name: frontend-rendering-debug
description: '除錯前端渲染問題，例如文字重疊、code block 顯示異常、CSS regression，或本地與正式環境渲染不一致。當頁面部署後顯示錯誤，且你需要用逐步流程找出控制該行為的 CSS 或 HTML 並套用最小安全修正時使用。'
argument-hint: '你正在排查哪一個渲染問題？'
user-invocable: true
---

# 前端渲染除錯

## 這個 Skill 會產生什麼
- 對部署頁面渲染問題的一次聚焦診斷
- 在實際控制該問題的 CSS 或 markup 層做最小可行修正
- 一次可確認修正命中正確範圍且 build 正常的驗證

## 何時使用
- 文字重疊、擠壓，或間距顯示異常
- code block、table、list 或 blockquote 在部署後外觀錯誤
- 部署頁面與本地預期顯示不同
- Hugo 或其他靜態站頁面在 Markdown 內容看起來正常，但渲染成 HTML/CSS 後錯誤

## 流程
1. 從部署頁面或明確壞掉的區塊開始，不要先做大範圍 repo 探索。
2. 先辨識最小且可見的症狀。
   例如：code block 文字重疊、line height 塌掉、換行錯誤、標題重複、table 版面壞掉。
3. 讀取最近的控制來源。
   先看受影響區段的內容檔，再看最接近且可能控制它的 stylesheet。
   在這個 repo 裡，優先看 `assets/ananke/css/custom.css`，再考慮更廣泛的 theme 檔案。
4. 判斷問題是內容驅動還是樣式驅動。
   如果原始 Markdown 結構本身就錯，先修內容。
   如果 Markdown 合理但渲染後畫面壞掉，就檢查 CSS 與實際 DOM。
5. 直接檢查部署頁面。
   取得壞掉區塊周圍的實際 HTML/CSS。
   優先看真正出問題的 `pre`、`code`、table、list 或 blockquote 元素。
6. 檢查受影響元素的 computed styles。
   至少檢查：`font-family`、`font-size`、`line-height`、`letter-spacing`、`white-space`、`word-break`、`overflow-wrap`、`display`，以及任何 transform。
7. 建立一個可被證偽的局部假設。
   例如：
   - code block 繼承了不合適的字型或 line-height
   - 某個換行規則讓 CJK 與 monospace 混排時字形碰撞
   - 某個全站 selector 意外影響了文章內容區
8. 在實際擁有控制權的層級套用最小修正。
   優先用 scoped selector，不要先做大範圍 global reset。
   以文章 code block 為例，可優先考慮：
   - target `.nested-copy-line-height pre`
   - target `.nested-copy-line-height pre code`
   - 明確設定穩定的 monospace 字型、line height、換行與 ligature 行為
9. 在第一次實質修改後立即驗證。
   跑最窄且最相關的 build 或 preview 驗證。
   在這個 repo 裡，使用 `hugo`。
10. 確認修正真的進到產出結果。
   驗證修改後的 CSS 規則是否出現在產出檔，或壞掉區塊是否已套用預期 computed style。
11. 當症狀解掉且 build 通過後就停止。
   不要順手做無關的 theme 清理。

## 決策點

### 內容 vs CSS
- 如果 Markdown 結構本身壞掉，先修內容。
- 如果部署後的 HTML 結構正常但顯示錯誤，修 CSS。

### 本地 vs 部署不一致
- 如果只有正式環境壞掉，先檢查部署產物與 production 專屬 CSS 行為。
- 如果本地與正式環境都壞掉，就一起檢查 source CSS 與內容結構。

### 修正範圍
- 如果只影響單一文章面，優先用 article-scoped selector。
- 如果多頁共用同一個症狀，就修共享樣式層。

## 品質檢查
- 修正已限制在最小相關 selector 範圍
- 站台 build 可以成功完成
- 目標區塊不再重疊或塌陷
- 窄寬度下的字體與間距仍可閱讀
- 除非 repo 明確要求追蹤 build 產物，否則不要提交無關的產出檔

## Repo 專屬備註
- 自訂呈現覆寫主要放在 `assets/ananke/css/custom.css`
- 站台可用 `hugo` 驗證
- 如果 repo 不打算追蹤產出檔，除非明確需要，否則不要提交 `public/` 與 `resources/_gen/`

## 範例 Prompt
- `/frontend-rendering-debug 修掉部署後 code block 文字重疊的問題`
- `/frontend-rendering-debug 調查這篇 Hugo 文章為什麼部署後看起來不一樣`
- `/frontend-rendering-debug 找出造成文章內容換行錯誤的 CSS`
