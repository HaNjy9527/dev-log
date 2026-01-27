+++
date = '2026-01-27T10:06:59+08:00'
draft = false
title = 'Hello World GitHub Pages + Hugo 建置開發日誌'
+++
## 這是我的第一篇網誌
這是一份完整的開發日誌摘要，你可以將其儲存為 `README.md` 或 `DEV_LOG.md` 放進你的專案中，方便未來查閱。

---

# GitHub Pages + Hugo 建置開發日誌

## 1. 專案概述

* **目標**：建立個人技術網誌 (`dev-log`)。
* **架構**：GitHub Pages (Hosting) + Hugo (Static Site Generator)。
* **主題**：Ananke。
* **自動化**：GitHub Actions。

---

## 2. 遭遇問題與解決方案 (Troubleshooting Log)

### 🔴 Git 上傳錯誤

**問題描述：**
執行 `git push -u origin main` 時出現以下錯誤：

```text
error: src refspec main does not match any
error: failed to push some refs to 'https://github.com/HaNjy9527/dev-log.git'

```

**原因分析：**

1. 本地端尚未進行第一次 `commit`，導致分支 (Branch) 尚未建立。
2. 本地預設分支名稱為 `master`，但 GitHub 預期為 `main`。

**解決指令：**

```bash
git add .
git commit -m "Initial commit"  # 建立實體版本
git branch -M main              # 強制更名為 main
git push -u origin main         # 推送

```

### 🔴 GitHub Actions 版本不相容

**問題描述：**
GitHub Actions 執行失敗，錯誤訊息：

```text
WARN Module "ananke" is not compatible with this Hugo version: Min 0.146.0

```

**原因分析：**
Ananke 主題要求 Hugo 版本至少為 `0.146.0`，但 GitHub Actions 環境預設安裝的是舊版。

**嘗試修正 (失敗)：**
將 `HUGO_VERSION` 改為 `"latest"`。

* **結果**：導致 `wget` 下載失敗 (404 Not Found)，因為下載網址不支援 `vlatest` 這種寫法。

**最終修正 (成功)：**
在 `.github/workflows/hugo.yml` 中指定明確的版本號。

```yaml
env:
  HUGO_VERSION: "0.146.0"  # 明確指定版本，解決 URL 404 與主題相容性問題

```

---

## 3. 常用操作指令備忘 (Cheatsheet)

### 📝 撰寫新文章

```bash
# 1. 建立新檔
hugo new posts/my-new-post.md

# 2. 本地預覽 (包含草稿)
hugo server -D

```

### 🚀 發布流程 (Deploy)

只要將 Markdown 檔案 Push 到 GitHub，網站就會自動更新。

```bash
git add .
git commit -m "新增文章: 標題"
git push

```

---

## 4. 環境設定紀錄

* **Repo URL**: `https://github.com/HaNjy9527/dev-log.git`
* **CI/CD Config**: `.github/workflows/hugo.yml`
* **Theme**: Ananke (as submodule)

---

你可以直接複製以上內容。如果之後還有遇到新問題，也可以繼續追加在這個檔案裡！