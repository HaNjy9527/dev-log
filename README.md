# Miguel Dev Log

Hugo 部落格專案，正式站點：<https://hanjy9527.github.io/dev-log/>

## 重點

- 靜態網站產生器：Hugo
- 佈景主題：Ananke（Git submodule）
- 文章位置：`content/posts/`
- 自訂樣式：`assets/ananke/css/custom.css`
- 產出目錄：`public/`

## 本機開發

先確認已安裝 Hugo extended，再於專案根目錄執行：

```bash
hugo server -D
```

預設可在 `http://localhost:1313/` 或 Hugo 命令列顯示的本機網址預覽。

## 新增文章

可直接在 `content/posts/` 新增 Markdown，或使用：

```bash
hugo new posts/my-post.md
```

## 部署

推送到 `main` 後，會由 GitHub Actions 自動建置並部署到 GitHub Pages。
