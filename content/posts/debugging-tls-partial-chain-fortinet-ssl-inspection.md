+++
date = '2026-05-04T00:00:00+08:00'
draft = false
title = '金流全斷的那個週日晚上：一次 TLS PartialChain 排查實錄'
categories = ["日誌", "除錯"]
tags = ["TLS", "SSL", "Fortinet", ".NET", "PowerShell", "Post-Mortem"]
description = '從應用程式 log 一路追到防火牆設定，記錄一次跨層級的 TLS 信任鏈排查過程，以及途中走過的彎路與學到的教訓。'
+++

這篇文章整理一次從應用程式錯誤一路追到網路出口設備的 TLS 排查過程，重點放在判斷順序、決定性驗證，以及最後如何收斂到可執行的處置方案。

<!--more-->

# 金流全斷的那個週日晚上：一次 TLS PartialChain 排查實錄

週日晚上九點半，剛回到家準備耍廢，下午同事傳來的訊息突然變得很燙手：

> 「線上金流全斷了，所有交易都進不去。」

接下來四個小時的排查過程，我從「以為是程式哪裡寫錯」開始，繞了好幾圈本機憑證庫、proxy 設定、TLS 版本，最後才在一個簡單的 5 行 PowerShell 指令前一秒看清真相——**根本不是程式問題，是公司防火牆做了 SSL Deep Inspection，但設備的 CA 從來沒部署到伺服器上**。

這篇文章是這次排查的完整記錄，包含我**做對的判斷、走錯的路、最後讓問題塵埃落定的那個決定性測試**，以及如果再遇到一次，我會怎麼做更快。

---

## 一、症狀

伺服器的 `nlog-temp-Error.log` 大量出現這類錯誤：

```
LinePay PayAsync - Exception:
  The SSL connection could not be established
  ---> System.Security.Authentication.AuthenticationException:
       The remote certificate is invalid because of errors
       in the certificate chain: PartialChain
   at System.Net.Security.SslStream.CompleteHandshake(...)
   at Car1.LinePay.Service.LinePayOnlineService.ProviderRequestAsync(...)
```

關鍵字 `PartialChain`——意思是「對方憑證鏈不完整，我接不到我信任的根」。

不只一個服務出事：

| 服務 | 錯誤型態 |
|---|---|
| 金流 A（直連 HTTPS） | `PartialChain` TLS 握手失敗 |
| 金流 B（直連 HTTPS） | `PartialChain` TLS 握手失敗 |
| 通訊軟體通報（直連 HTTPS） | `SSL connection could not be established` |
| 金流 C（走本機 SDK） | provider 回傳業務錯誤碼（**不同根因，後文解釋**） |

時間點很乾淨：**前一天晚上 8:30 開始全部斷線**，前後沒有任何部署或設定變更。

---

## 二、排查歷程：從程式碼一路追到防火牆

### 1. 先把「真錯誤」和「噪音」分開

第一個小時花在分類 log。發現有三類訊號混在一起：

1. 一些「臨停明細查無紀錄」這類錯誤——**走的是內網 HTTP，跟 TLS 無關**，先排除
2. 一個 `argumentException: url is null`——**程式邏輯漏防呆**，但不影響金流
3. **TLS 失敗**——這條才是金流真正斷掉的原因

**這一步排除的可能：** 商業邏輯錯誤、API 參數錯誤、內網資料問題。

### 2. 是「單一服務」還是「整台機器都壞」？

如果只有一個金流商失敗，就懷疑對方端；多個獨立服務同時失敗，就要懷疑環境。

| 證據 | 排除的可能 |
|---|---|
| 通訊軟體通報也 SSL 失敗 | 不是金流商單站問題 |
| 金流 C 仍能拿到 provider 回應 | 不是「整台機器網路全斷」 |
| 金流 A、B 都出 PartialChain | 至少有兩條對外 HTTPS 同型失敗 |

**這一步排除的可能：** 單一供應商問題、整機網路完全失效。

### 3. TCP / TLS / 協定三層獨立測試

用最便宜的方式分層驗證。

```powershell
# Layer 1: TCP 通不通？
Test-NetConnection api-pay.example.com -Port 443
# Result: TcpTestSucceeded : True
```
TCP 通。

```powershell
# Layer 2: HTTPS 握手成不成?
Invoke-WebRequest https://api-pay.example.com -UseBasicParsing
# Result: 基礎連接已關閉: 傳送時發生未預期的錯誤
```
卡在 TLS 握手。

```powershell
# Layer 3: 強制 TLS 1.2 排除版本問題
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Invoke-WebRequest https://api-pay.example.com -UseBasicParsing
# Result: 基礎連接已關閉: 無法為 SSL/TLS 安全通道建立信任關係
```

訊息明確指向「**信任關係**」——憑證鏈問題，不是 TLS 版本問題。

```powershell
# 順手確認 proxy 設定
netsh winhttp show proxy
# Result: 直接存取 (不使用 Proxy 伺服器)
```

沒有顯式 proxy。

走到這裡我已經很有信心是**機器層的憑證信任鏈問題**，但**還不知道是「本機庫缺東西」還是「中間有設備搞鬼」**。

### 4. 本機憑證庫檢查：這裡走偏了

我打開 `certlm.msc`，仔細檢查：

- 受信任的根憑證授權單位 → 常見公開 CA 都有
- 中繼憑證授權單位 → 看起來正常
- 不受信任的憑證 → 沒有可疑封鎖

**結論：本機憑證庫看起來沒明顯異常 → 我以為方向錯了。**

> ⚠️ **這是整個排查最大的時間浪費點。** 看 GUI 看了 40 分鐘，得到一個「看似沒問題」的結論，但問題其實壓根不在這裡。
>
> 後來才意識到：**本機憑證庫看起來再正常都沒用——因為實際握手收到的根本不是「正常路徑上會看到」的憑證**。我看的是地圖上的 A 點，但問題在 B 點。

### 5. 抓取遠端實際送回的憑證：關鍵一步

第三個小時，我終於想到應該直接看「**對方丟給我什麼**」，而不是猜「**我這邊缺什麼**」。

用 `SslStream` 跳過信任驗證、把握手過程中對方送來的憑證原汁原味讀出來：

```powershell
$hostName = "api-pay.example.com"
$tcp = New-Object System.Net.Sockets.TcpClient($hostName, 443)
$callback = [System.Net.Security.RemoteCertificateValidationCallback]{ $true }
$ssl = New-Object System.Net.Security.SslStream($tcp.GetStream(), $false, $callback)
$ssl.AuthenticateAsClient($hostName, $null, [System.Security.Authentication.SslProtocols]::Tls12, $false)
$ssl.RemoteCertificate.Issuer
```

結果：

```
Issuer: E=support@<vendor>.com, CN=<DEVICE-SERIAL>,
        OU=Certificate Authority, O=<NetworkVendor>,
        L=Sunnyvale, S=California, C=US
```

**那一刻所有疑問瞬間解開**：

- 我們連的是金流 API，但收到的憑證 Issuer 是**公司的網路設備廠商**
- 這代表中間有設備在做 SSL Deep Inspection（解開來看內容、再用自簽憑證重新加密）
- 而我們的伺服器**沒有部署過這個設備的 CA** → 信任鏈當然斷掉

**「為什麼不可能是程式問題」的鐵證也在這裡誕生**：對方送回來的憑證 Issuer 跟程式碼一行都無關，是在我們網路出口的某個設備上發生的事。

---

## 三、最終驗證：從「強烈懷疑」到「100% 確定」

雖然抓到偽憑證已經很有說服力，但我想再做兩個確認，把所有可能的反駁都堵死。

### 驗證 1：本機真的沒有這張 CA 嗎？

```powershell
$stores = @(
    "Cert:\CurrentUser\Root", "Cert:\CurrentUser\CA", "Cert:\CurrentUser\AuthRoot",
    "Cert:\LocalMachine\Root", "Cert:\LocalMachine\CA", "Cert:\LocalMachine\AuthRoot",
    "Cert:\LocalMachine\TrustedPublisher", "Cert:\LocalMachine\Trust"
)

foreach ($store in $stores) {
    $found = Get-ChildItem $store -ErrorAction SilentlyContinue | Where-Object {
        $_.Subject -like "*<vendor>*" -or $_.Issuer -like "*<vendor>*"
    }
    if ($found) { Write-Host "$store : FOUND" }
    else        { Write-Host "$store : (none)" }
}
```

8 個存放區全部 `(none)`——**整台機器完全沒有這個廠商的 CA**。

### 驗證 2：攔截範圍有多大？

我把測試函式化，跑一輪業界常見的網域：

```powershell
function Test-CertIssuer($hostName) {
    try {
        $tcp = New-Object System.Net.Sockets.TcpClient($hostName, 443)
        $cb = [System.Net.Security.RemoteCertificateValidationCallback]{ $true }
        $ssl = New-Object System.Net.Security.SslStream($tcp.GetStream(), $false, $cb)
        $ssl.AuthenticateAsClient($hostName, $null,
            [System.Security.Authentication.SslProtocols]::Tls12, $false)
        Write-Host "$hostName -> $($ssl.RemoteCertificate.Issuer)" -ForegroundColor Cyan
        $ssl.Close(); $tcp.Close()
    } catch {
        Write-Host "$hostName -> ERROR: $($_.Exception.Message)" -ForegroundColor Red
    }
}

@(
    "google.com", "github.com",
    "api-pay.example.com", "another-payment.example.com",
    "<other-vendors>"
) | ForEach-Object { Test-CertIssuer $_ }
```

結果：**所有目標網域，包含 google.com 和 github.com，回傳的 Issuer 全部都是同一張偽憑證**。

這代表攔截範圍是**全公司網段、所有對外 HTTPS 流量，沒有任何例外**。

### 三個事實交叉驗證

| 驗證 | 結果 | 推論 |
|---|---|---|
| 抓到的憑證 Issuer | 全部 = 廠商自簽 | 出口被 SSL Inspection 攔截 |
| 本機廠商 CA | 完全沒有 | 即使知道是誰簽的也信不過 |
| 攔截範圍 | 全部主機 | 是全網段政策，非選擇性 |

**根因到此 100% 確定。**

---

## 四、但等等：為什麼瀏覽器看起來沒事？

這是一個讓我一度困惑的細節。同一台伺服器上 RDP 開 Chrome 連 google.com、github.com 都能正常使用，**為什麼只有 .NET 應用程式炸**？

兩個原因：

1. **Chrome 從 v105 開始用自己內建的 Chrome Root Store**，不再完全依賴 Windows 憑證庫
2. **更可能的原因**：曾經有人（可能是前任工程師遠端進來）按過「**進階 → 繼續前往（不安全）**」，Chrome 記住了 host exception。網址列其實有「不安全」紅色警告，只是平常沒人注意

而 .NET / Schannel **完全不會忽略警告**——遇到不信任的憑證直接擲例外，金流就掛了。

**結論：瀏覽器能用 ≠ 應用程式能用。** 用瀏覽器測試只能證明「網路有通」，不能證明「應用程式也能通」。

---

## 五、為什麼是那個時間點：小謎團

排完根因之後，我還是很好奇「**為什麼偏偏是那天晚上 8:30 突然發作**」。如果攔截早就在做，為什麼不是更早爆？

我重抓一次偽憑證，看 NotBefore：

```powershell
$cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2($ssl.RemoteCertificate)
$cert | Format-List Subject, Issuer, NotBefore, NotAfter, Thumbprint
```

結果 NotBefore 是**九個多月前**——不是新換的 CA，所以「CA 輪替」這個劇本被排除。

最後跨部門對話確認後，最可能的劇本是：**設備的雲端評級服務在那天晚上出狀況，導致原本被分類為「金融類 → bypass」的網域分類失效，全部變成「未知類 → 強制 Inspection」**。換句話說，攔截功能一直在跑，只是之前金流網域有 bypass 規則，那天晚上 bypass 規則因為分類查不到而失效。

但說實在的，這個「為什麼」對解決方案沒有任何影響——**不管哪個劇本，處置方式都一樣**。我給自己留下這個未解之謎，不再深挖。

---

## 六、處置：bypass vs 部署 CA

技術上有兩個方向，**強烈建議並行**：

### 方案 1：把金流網域加進 SSL Inspection bypass（短期）

讓 FortiGate 直接放行金流流量，不解密、不換憑證。優點：

- 立即恢復
- **金流流量不被中間設備解密**——符合 PCI-DSS 思維
- 未來金流商若啟用 Certificate Pinning 也不會踩雷

### 方案 2：透過 GPO 部署設備 CA 到所有伺服器（中期）

讓所有應用程式信任設備的偽憑證。優點：

- 涵蓋面廣（所有 HTTPS 都受惠）
- 解決其他應用程式的潛在問題（Windows Update、套件管理、第三方 SDK）

### 為什麼兩個都要做

| 面向 | 方案 1 (bypass) | 方案 2 (部署 CA) |
|---|---|---|
| 解金流問題 | ✅ | ✅ |
| 解未來新外部服務 | ❌ 每次要再開單 | ✅ 全自動 |
| Certificate Pinning 風險 | ✅ 安全 | ❌ 高風險 |
| PCI-DSS 合規 | ✅ 安全 | ❌ 不利稽核 |

**單做方案 2 不做方案 1 的風險**：金流商若啟用 Certificate Pinning，部署 CA 也救不回來。

---

## 七、學到的教訓：下次怎麼更快

### 1. 看到 `PartialChain` 就應該立刻往「環境層」查

`PartialChain` 永遠是**憑證鏈建不起來**，幾乎不可能是程式碼問題。下次看到這個關鍵字，**第一動作直接跳到「抓遠端憑證 Issuer」**，不要先讀程式碼、不要先查 HttpClient 設定。

### 2. 「抓遠端實際送回的憑證」應該是 step 1，不是 step N

整個流程中我走過：log 分析、程式碼審查、TCP 測試、TLS 版本測試、proxy 設定、本機憑證庫 GUI 檢查、事件檢視器……**最後才用 SslStream 抓憑證**。

但這一步其實是**最便宜、最快、訊息量最大的測試**：5 行 PowerShell、1 秒鐘、看一個欄位（Issuer），就能直接知道是不是被攔截。

下次的順序應該是：

```
1. 看 log 確認錯誤型態（PartialChain / Untrusted / Expired）
2. 用 SslStream 抓對方憑證的 Issuer ← 最早期就做
3. Issuer 對 → 查本機憑證鏈 / 中繼缺失
   Issuer 不對 → 查中間設備 / SSL Inspection / Proxy
4. 確認攔截範圍（單站 / 全網）
5. 找適合的處置方案
```

### 3. 多廠商同時故障 = 環境問題

兩個以上彼此獨立的第三方服務同時壞，且錯誤型態一致 → **應該立刻從應用層轉向環境層**。可以建立直覺：

> N 個獨立服務同時壞，N≥2 → 公司網路 / 機器環境問題  
> N=1 → 單一供應商或 endpoint 問題

### 4. 瀏覽器寬鬆，應用程式嚴格

| 客戶端 | 行為 |
|---|---|
| **瀏覽器** | 可手動忽略警告、自動補抓中繼憑證、有自己的 Root Store、對 timeout 自動 retry |
| **.NET / Java / Python** | 任何信任失敗直接擲例外、補抓中繼憑證較保守、服務帳戶用 LocalMachine 憑證庫（不一定跟使用者帳戶一樣） |

**用瀏覽器當基準會誤判**——它就是寬鬆過頭。

### 5. 「沒有 proxy」不等於「沒有中間設備」

`netsh winhttp show proxy` 顯示「直接存取」很容易讓人放心：

- ✅ 它能排除：顯式設定的 WinHTTP proxy
- ❌ 它無法排除：透明代理、SSL Inspection、出口閘道、EDR、防火牆 HTTPS 解密

下次遇到類似問題時，**「沒有 proxy」這個結論要小心解讀**。

---

## 八、副產品：環境健康檢查腳本

這次最有長期價值的不是解掉問題，是順手做出來的這個小工具。存成 `check-tls.ps1`，**未來只要金流又出事，第一件事就是執行它**：

```powershell
function Test-CertIssuer($hostName, $port = 443) {
    try {
        $tcp = New-Object System.Net.Sockets.TcpClient($hostName, $port)
        $cb = [System.Net.Security.RemoteCertificateValidationCallback]{ $true }
        $ssl = New-Object System.Net.Security.SslStream($tcp.GetStream(), $false, $cb)
        $ssl.AuthenticateAsClient($hostName, $null,
            [System.Security.Authentication.SslProtocols]::Tls12, $false)
        $issuer = $ssl.RemoteCertificate.Issuer
        $org = if ($issuer -match 'O=([^,]+)') { $matches[1] } else { $issuer }
        Write-Host "$hostName -> $org" -ForegroundColor Cyan
        $ssl.Close(); $tcp.Close()
    } catch {
        Write-Host "$hostName -> ERROR: $($_.Exception.Message)" -ForegroundColor Red
    }
}

# 一次測一批（依你的場景客製）
@(
    "google.com",                    # 控制組
    "github.com",                    # 控制組
    "api-pay.example.com",           # 你關心的金流
    "another-payment.example.com"
) | ForEach-Object { Test-CertIssuer $_ }
```

正常環境跑出來會像這樣：

```
google.com -> Google Trust Services
github.com -> Sectigo Limited
api-pay.example.com -> GlobalSign nv-sa
another-payment.example.com -> Sectigo Limited
```

如果有任何一行 Issuer 出現公司防火牆廠商名稱，就知道又被攔了。**5 秒看完，比讀 log 快太多**。

---

## 九、五句話帶走

1. **`PartialChain` 是憑證鏈問題，不是程式碼問題。**
2. **抓遠端實際送回的憑證，5 行 PowerShell 勝過 5 小時的 log 分析。**
3. **多廠商同時故障 = 環境問題，幾乎沒有例外。**
4. **瀏覽器能用不代表應用程式能用，因為瀏覽器寬鬆、Schannel 嚴格。**
5. **「沒有 proxy」≠「沒有中間設備」**，透明代理永遠不會出現在 `netsh winhttp show proxy` 裡。

---

## 十、後記

整個排查從晚上 9:30 開始，到凌晨快 1 點告一段落，傳完訊息給主管後就去睡了。隔天早上跨部門對接，下午金流恢復。

整個事件最讓我有感觸的不是技術，是**判斷「什麼時候該深挖、什麼時候該 ship」這件事**。我大可以追究 100% 的根因（為什麼是那個時間點？評級服務怎麼出狀況的？），但金流恢復是首要目標，根因追究是 nice-to-have。**工程師的價值不只是技術，更是判斷力**。

希望這篇對下一個踩到 `PartialChain` 的工程師有點用。
