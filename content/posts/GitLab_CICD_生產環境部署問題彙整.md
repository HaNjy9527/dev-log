
+++
date = '2026-01-27T10:10:00+08:00'
draft = false
title = 'GitLab CI/CD 生產環境部署問題彙整'
categories = ["CI/CD", "部署"]
tags = ["GitLab", "Runner", "IIS", "PowerShell", ".NET"]
+++

這篇文章整理了 GitLab CI/CD 在生產環境部署時常見的問題與對策，包含 Runner、權限、IIS 與 .NET Hosting Bundle 等實務細節。

# GitLab CI/CD 生產環境部署問題彙整報告

## 一、環境基礎建設問題：找不到 Git

### 狀況描述
在生產伺服器執行 `get_sources` 階段時報錯 `The term 'git' is not recognized`。

### 原因分析
1. 生產伺服器尚未安裝 Git 客戶端
2. 安裝後未將 Git 路徑加入系統環境變數（Path），或 Runner 未重啟導致讀取不到新變數
<!--more-->
### 解決方式
- 在伺服器安裝 Git
- 將 `C:\Program Files\Git\bin` 加入系統 Path
- **關鍵動作**：執行 `Restart-Service gitlab-runner` 重新載入環境變數

---

## 二、檔案部署邏輯問題：Copy-Item 失敗

### 狀況描述
出現錯誤 `Container cannot be copied onto existing leaf item`。

### 原因分析
1. `Copy-Item` 在 PowerShell 處理「資料夾對資料夾」的覆蓋時邏輯較脆弱，若目標路徑已存在同名檔案或結構不符會報錯
2. 檔案被 IIS 程序佔用導致無法覆蓋

### 解決方式
- 改用 `robocopy`：Windows 內建更強大的同步工具
- 參數優化：使用 `/E` (子目錄)、`/MT:32` (多執行緒)、`/R:2 /W:5` (失敗重試機制)
- 能有效解決暫時性的檔案鎖定問題

---

## 三、IIS 自動化控制問題：應用程式集區狀態報錯

### 狀況描述
執行 `Stop-WebAppPool` 時出現 `目標路徑上的物件已經停止`，導致 Pipeline 中斷（ERROR: Job failed）。

### 原因分析
1. IIS 指令在目標狀態已達成時（本來就停止了）會拋出 Exception
2. 在 PowerShell Core (pwsh) 執行舊版 `WebAdministration` 模組時，錯誤攔截機制較敏感

### 解決方式（演進過程）

#### 方案一：Try-Catch 攔截（Gemini 建議）
```powershell
try {
    Stop-WebAppPool -Name $env:IIS_APP_POOL
} catch {
    if ($_.Exception.Message -like "*已經停止*") {
        Write-Host "應用程式集區已經是停止狀態"
    }
}
```
**問題**：在 GitLab CI YAML 中使用多行區塊時遇到語法錯誤

#### 方案二：先檢查狀態再操作（Claude 最終方案）
```powershell
$appPool = Get-WebAppPoolState -Name $env:IIS_APP_POOL
if ($appPool.Value -eq "Started") {
    Stop-WebAppPool -Name $env:IIS_APP_POOL
    Start-Sleep -Seconds 3
    Write-Host "已停止應用程式集區"
} else {
    Write-Host "應用程式集區已經是停止狀態"
}
```
**優點**：更簡潔、避免錯誤、邏輯清晰

---

## 四、GitLab CI YAML 語法問題

### 狀況描述
出現錯誤 `jobs:deploy:script config should be a string or a nested array of strings up to 10 levels deep`

### 原因分析
1. YAML 的多行區塊標記 `|` 在 GitLab CI 中對 PowerShell 腳本支援不佳
2. 嵌套的 if-else 和 try-catch 結構造成層級過深

### 解決方式
- 移除所有 `|` 多行區塊標記
- 將多行邏輯改為單行，使用 `;` 分隔命令
- 複雜邏輯使用 `{}` 包裹，放在同一行

---

## 五、Robocopy 退出代碼問題

### 狀況描述
Robocopy 成功複製檔案（退出代碼 1），但 GitLab CI 將非零退出代碼視為失敗，導致後續步驟未執行。

### 原因分析
1. Robocopy 的退出代碼 0-7 都表示成功，8 以上才是錯誤
2. GitLab Runner 在看到非零退出代碼後立即中斷，未執行後續的退出代碼檢查邏輯

### 解決方式
將 robocopy 和退出代碼檢查合併為一行：
```powershell
robocopy ... ; if ($LASTEXITCODE -lt 8) { $LASTEXITCODE = 0 } else { exit 1 }
```

**Robocopy 退出代碼說明**：
- 0 = 沒有檔案被複製
- 1 = 成功複製檔案
- 2 = 有額外的檔案或目錄
- 3 = 有檔案被複製且有額外檔案
- 4 = 有不匹配的檔案或目錄
- 5-7 = 其他成功狀態的組合
- 8+ = 失敗

---

## 六、PowerShell 命令退出代碼問題

### 狀況描述
`Get-ChildItem | Remove-Item` 執行後返回非零退出代碼，導致 Pipeline 失敗。

### 原因分析
即使使用了 `-ErrorAction SilentlyContinue`，PowerShell 在某些情況下仍會返回非零退出代碼。

### 解決方式
在可能失敗的命令後明確重置退出代碼：
```powershell
Get-ChildItem ... | Remove-Item ... ; $LASTEXITCODE = 0
```

---

## 七、安全與設定保護問題：設定檔被覆蓋

### 狀況描述
部署時會把生產環境特有的 `appsettings.Production.json` 或 `web.config` 刪除或覆蓋。

### 解決方式
- 在 `Get-ChildItem` 清理階段加入 `-Exclude` 參數
- 在 `robocopy` 階段加入 `/XF` (排除檔案) 與 `/XD` (排除資料夾，如 Logs、Uploads、wwwroot)
```powershell
# 清理時排除
Get-ChildItem -Path $env:IIS_SITE_PATH -Exclude appsettings.Production.json,web.config,Logs,Uploads,wwwroot | Remove-Item -Recurse -Force

# Robocopy 時排除
robocopy ... /XF appsettings.Production.json web.config /XD Logs Uploads wwwroot
```

---

## 🛠️ 最終推薦的部署流程架構

| 階段 (Stage) | 執行動作 (Action) | 核心指令 / 技巧 |
|-------------|------------------|----------------|
| Build | 編譯並發布 Artifacts | `dotnet publish` |
| Deploy | 檢查並停止 IIS AppPool | `Get-WebAppPoolState` → 條件性 `Stop-WebAppPool` |
| Deploy | 清理舊程式碼（保留資料） | `Remove-Item -Exclude ...` + `$LASTEXITCODE = 0` |
| Deploy | 執行增量部署 | `robocopy /E /XF /XD` + 立即重置退出代碼 |
| Deploy | 重新啟動 AppPool | `Start-WebAppPool` |

---

## 💡 關鍵經驗總結

### 1. 容錯機制優先
生產環境充滿不可預測的狀態（檔案鎖定、集區已停止、路徑差異），**預先檢查狀態**比**事後捕獲錯誤**更可靠。

### 2. 工具選擇
- `robocopy` > `Copy-Item`：重試機制、多執行緒、更好的檔案鎖定處理
- `Get-WebAppPoolState` 檢查 > `try-catch` 捕獲：邏輯更清晰、避免錯誤

### 3. GitLab CI 特殊性
- 避免複雜的多行區塊和深層嵌套
- 關鍵命令後立即處理退出代碼
- 使用 `;` 將相關命令合併為原子操作

### 4. Gemini vs Claude 的差異
- **Gemini**：提供了 try-catch 方案，但未考慮 GitLab CI YAML 語法限制
- **Claude**：直接採用更簡潔的「先檢查再操作」方案，並解決了 YAML 語法和退出代碼問題

---

## 📋 最終可運行的完整 YAML 範例
```yaml
stages:
  - build
  - deploy

variables:
  PROJECT_PATH: "MyWebApp"
  PROJECT_NAME: "MyWebApp"
  PUBLISH_PATH: "./publish"
  IIS_SITE_PATH: "C:\\inetpub\\wwwroot\\MyWebApp"
  IIS_SITE_NAME: "MyWebApp"
  IIS_APP_POOL: "MyWebAppPool"
  DEPENDENCY_REPO: "http://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.example.com/team/dependency-lib.git"
  DEPENDENCY_REF: "master"

build:
  stage: build
  tags:
    - windows
  script:
    - if (Test-Path "..\DependencyLib") { git -C "..\DependencyLib" remote set-url origin $env:DEPENDENCY_REPO }
    - if (!(Test-Path "..\DependencyLib")) { git clone --branch $env:DEPENDENCY_REF $env:DEPENDENCY_REPO "..\DependencyLib" }
    - git -C "..\DependencyLib" fetch --all
    - git -C "..\DependencyLib" checkout $env:DEPENDENCY_REF
    - git -C "..\DependencyLib" pull
    - cd $env:PROJECT_PATH
    - dotnet restore
    - dotnet build --configuration Release --no-restore
    - cd ..
    - dotnet publish "$env:PROJECT_PATH/$env:PROJECT_NAME.csproj" --configuration Release --output $env:PUBLISH_PATH
    - Get-ChildItem -Path $env:PUBLISH_PATH
  artifacts:
    paths:
      - $PUBLISH_PATH
    expire_in: 1 hour
  only:
    - main

deploy:
  stage: deploy
  tags:
    - windows
  script:
    - Import-Module WebAdministration
    - Write-Host "處理應用程式集區 $env:IIS_APP_POOL"
    - $appPool = Get-WebAppPoolState -Name $env:IIS_APP_POOL
    - if ($appPool.Value -eq "Started") { Stop-WebAppPool -Name $env:IIS_APP_POOL; Start-Sleep -Seconds 3; Write-Host "已停止應用程式集區" } else { Write-Host "應用程式集區已經是停止狀態" }
    - Write-Host "清理舊檔案"
    - Get-ChildItem -Path $env:IIS_SITE_PATH -Exclude appsettings.Production.json,web.config,Logs,Uploads,wwwroot | Remove-Item -Recurse -Force -ErrorAction SilentlyContinue; $LASTEXITCODE = 0
    - Write-Host "部署新版本"
    - robocopy "$env:PUBLISH_PATH" "$env:IIS_SITE_PATH" /E /MT:32 /R:2 /W:5 /XF appsettings.Production.json web.config /XD Logs Uploads wwwroot; if ($LASTEXITCODE -lt 8) { $LASTEXITCODE = 0 } else { exit 1 }
    - Write-Host "複製完成"
    - Write-Host "啟動應用程式集區"
    - Start-WebAppPool -Name $env:IIS_APP_POOL
    - Write-Host "部署完成"
  dependencies:
    - build
  only:
    - main
```

---

## 📌 下一步建議

### 1. 環境變數外部化
將敏感或環境特定的變數移到 GitLab Project Settings → CI/CD → Variables：
- `IIS_SITE_PATH`
- `IIS_APP_POOL`
- `DEPENDENCY_REPO`（如果包含認證資訊）

**優點**：
- 提高安全性（不會出現在程式碼倉庫中）
- 更容易管理多環境部署（dev、staging、production）
- 避免敏感資訊洩露

### 2. 增加回滾機制
```powershell
# 部署前備份
$backupPath = "C:\Backups\MyWebApp_$(Get-Date -Format 'yyyyMMdd_HHmmss')"
robocopy $env:IIS_SITE_PATH $backupPath /E

# 如果部署失敗，可以快速回滾
```

### 3. 健康檢查
```powershell
# 部署完成後檢查
Start-Sleep -Seconds 5
$response = Invoke-WebRequest -Uri "http://localhost/health" -UseBasicParsing
if ($response.StatusCode -ne 200) {
    Write-Error "健康檢查失敗"
    exit 1
}
```

### 4. 通知機制
整合 Slack/Email/Teams 通知部署結果，在 `deploy` stage 最後加入：
```yaml
after_script:
  - |
    if ($CI_JOB_STATUS -eq "success") {
        # 發送成功通知
    } else {
        # 發送失敗通知
    }
```

### 5. 日誌保留
```powershell
# 保留最近 N 個版本的備份
Get-ChildItem "C:\Backups" | 
    Where-Object { $_.Name -like "MyWebApp_*" } | 
    Sort-Object LastWriteTime -Descending | 
    Select-Object -Skip 5 | 
    Remove-Item -Recurse -Force
```

---

## 🔍 故障排查清單

當部署失敗時，按以下順序檢查：

1. **檢查 GitLab Runner 服務是否運行**
```powershell
   Get-Service gitlab-runner
```

2. **檢查 IIS 應用程式集區狀態**
```powershell
   Get-WebAppPoolState -Name "MyWebAppPool"
```

3. **檢查檔案是否被鎖定**
```powershell
   # 使用 Process Explorer 或 Handle 工具
   handle.exe C:\inetpub\wwwroot\MyWebApp
```

4. **檢查磁碟空間**
```powershell
   Get-PSDrive C
```

5. **查看詳細的 Robocopy 日誌**
```powershell
   # 在 robocopy 命令加入 /LOG 參數
   robocopy ... /LOG:C:\Logs\robocopy.log
```

---

## 📚 參考資源

- [GitLab CI/CD YAML 語法](https://docs.gitlab.com/ee/ci/yaml/)
- [Robocopy 完整參數說明](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/robocopy)
- [PowerShell WebAdministration 模組](https://learn.microsoft.com/en-us/powershell/module/webadministration/)
- [GitLab Runner Windows 安裝指南](https://docs.gitlab.com/runner/install/windows.html)

---

**文件版本**：1.0  
**最後更新**：2026-01-27  
**維護者**：我本人 + Claude + Gemini（AI 協作除錯小組）  
**備註**：本文件是血淚史的結晶，每個問題都是真實踩過的坑 💀

**特別感謝**：  
- GitLab CI 的各種神秘錯誤訊息，讓我們有機會寫這份文件  
- Robocopy 的非零退出代碼，教會我們什麼叫「成功也會報錯」  
- PowerShell 的 `$LASTEXITCODE`，人生導師  

**心得**：當 Gemini 處理不好的時候，就換 Claude 上場 😎