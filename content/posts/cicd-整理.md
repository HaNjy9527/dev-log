+++
date = '2026-01-27T10:06:59+08:00'
draft = false
title = 'CI/CD 經歷整理（GitLab + Windows + IIS）'
categories = ["日誌", "CI/CD"]
tags = ["GitLab", "Runner", "Windows", "IIS", ".NET"]
+++

## 1. 目標與環境設定階段
**目標**：在公司自架 GitLab 上為 .NET 專案建立 CI/CD，部署到 IIS。

**環境與策略**
- Windows Server + IIS
- GitLab Runner 使用 Shell Executor（非 Docker）
- 先做 Pilot Run，確認穩定再推廣

**關鍵認知補強**
- GitLab Server 只負責管理流程，實際執行需要 GitLab Runner
- 公司自架 GitLab + 自架 Runner 本身免費（僅機器成本）

---
<!--more-->
## 2. Runner 安裝與註冊階段
**問題：** 無法執行 `gitlab-runner.exe`
- **原因**：檔名與位置不一致（原檔名為 `gitlab-runner-windows-amd64.exe`）
- **處理**：建立 `C:\GitLab-Runner`，重新命名並整理檔案

**問題：** Runner 註冊參數不確定
- **處理**：
  - Tags：`windows, shell`
  - Executor：`shell`
  - Maintenance note：略過

**結果：** Runner 成功安裝為 Windows 服務並啟動

---

## 3. CI 編譯與相依問題處理階段
**問題：** 找不到 PowerShell Core（`pwsh`）
- **原因**：Runner 預設用 `pwsh`，但伺服器只有 Windows PowerShell
- **處理**：安裝 PowerShell 7+，重啟 Runner 服務

**問題：** `gitlab-runner` 指令找不到
- **原因**：未加入 PATH
- **處理**：改以 Windows 服務方式管理或用完整路徑執行

**問題：** .gitlab-ci.yml 語法錯誤
- **原因**：縮排與註解造成 YAML 格式問題
- **處理**：移除中文註解、統一 2 空格縮排、重建檔案

**問題：** .NET SDK 版本不符（專案為 .NET 8）
- **原因**：伺服器僅安裝 .NET 6 SDK
- **處理**：安裝 .NET 8 SDK（不需重開機）

**問題：** 專案參考找不到（`Shared.Utils.csproj`）
- **原因**：相依專案未拉取或路徑錯誤
- **處理**：在 CI build 前自動 clone `SharedRepo` 專案並切到指定分支

**問題：** clone 失敗（`HTTP Basic Access denied`）
- **原因**：CI_JOB_TOKEN 無權限
- **處理**：在 `SharedRepo` 啟用 Token Access，並設定來源專案；必要時重設遠端 URL

**問題：** `SharedFeature` 命名空間不存在
- **原因**：相依功能在 `master` 分支
- **處理**：SharedRepo 固定使用 `master`

**問題：** publish 找不到 `Shared.Utils.dll`
- **原因**：`dotnet publish` 使用 `--no-build`，未產出 Release
- **處理**：移除 `--no-build`

---

## 4. CD 部署與 IIS 相關問題處理階段
**問題：** `Stop-WebAppPool` 找不到
- **原因**：未載入 WebAdministration 模組或未安裝 IIS 管理工具
- **處理**：在 deploy 加入 `Import-Module WebAdministration`

**問題：** 部署清理誤刪 Logs / Uploads
- **處理**：清理時排除 `Logs` 與 `Uploads`

**問題：** HTTP 500.19（IIS 無法識別 `<aspNetCore>`）
- **原因**：缺少 ASP.NET Core Module（Hosting Bundle）
- **處理**：安裝 .NET Hosting Bundle，並執行 `iisreset`

---

## 5. 目前 `.gitlab-ci.yml` 的關鍵策略
- build 前先拉 `SharedRepo`，並確保 token 與分支正確
- publish 會產生 Release 輸出
- deploy 載入 IIS 模組，並排除 Logs/Uploads

---

## 6. 總結（流程縮寫）
1. **理解架構**：Runner 必不可少
2. **安裝註冊**：Runner/Executor/Tags 正確配置
3. **CI 編譯**：處理 SDK、相依專案、YAML、Token 權限
4. **CD 部署**：IIS 模組、Hosting Bundle、清理策略
5. **穩定化**：分支鎖定、產出 Release、排除重要資料夾
