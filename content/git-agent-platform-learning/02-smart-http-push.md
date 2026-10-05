---
title: git push、Smart HTTP 與 receive-pack
description: 追蹤 push 從 HTTP ref discovery 到伺服器更新 refs 的流程。
tags:
  - git
  - smart-http
  - server
date: 2026-10-05
---

# git push、Smart HTTP 與 receive-pack

## Push 時伺服器大致做什麼

1. Git client 透過 Smart HTTP 對 **/info/refs?service=git-receive-pack** 做 ref discovery，取得伺服器目前的 refs 和支援能力。
2. client POST 到 **/git-receive-pack**，送出要更新的 ref、old/new object ID 和 packfile。
3. Web server 將請求交給 **git-http-backend**；backend 啟動 Git 的 **git-receive-pack** 服務處理 push。
4. receive-pack 將新 objects 放進 quarantine 暫存區，然後執行 pre-receive。若 hook 拒絕，ref 不更新，quarantine objects 會被移除。
5. 驗證通過後，Git 把 objects 移入主要 object store，執行 ref 更新並回傳每個 ref 的結果。

這不是說 git-http-backend 本身就是完整的網站伺服器。Web server 或 reverse proxy 還要處理 TLS、路由、驗證和存取權限。

## Pro Git 第 4 章原文整理

以下依 Pro Git 第二版英文第 4.1 與 4.6 節翻譯整理；重點放在協定差異和 Smart HTTP 的伺服器責任。它是摘要式筆記，不是逐段直譯。Pro Git 由 Scott Chacon 與 Ben Straub 撰寫，採 CC BY-NC-SA 3.0。

### 4.1 傳輸協定的取捨

遠端 repository 通常是 bare repository，只有 Git 資料，沒有 checkout 出來的工作目錄。Pro Git 將 Git 傳輸方式分成四類：

| 協定 | 主要特性 | 對自架平台的意義 |
| --- | --- | --- |
| Local | 透過本機路徑或共享檔案系統存取。簡單，沿用檔案權限；遠端使用時得先掛載檔案系統，也讓使用者直接接觸 repository 內部檔案。 | 適合本機或可信任的共享儲存，不是一般網際網路服務的入口。 |
| SSH | 傳輸加密並驗證連線，通常透過使用者帳號或 SSH key 執行 Git。常見、有效率，但不能提供匿名存取。 | 適合管理者熟悉 SSH 的自架環境；授權通常跟主機帳號或 key 管理結合。 |
| Git protocol（`git://`） | 由 Git daemon 提供專用傳輸，設計簡單且效率高；本身沒有認證或加密。 | 可供公開唯讀 clone；不能把未受保護的 `git://` 當成安全 push 入口。 |
| HTTP | 分成 Smart HTTP 與 Dumb HTTP。Smart HTTP 能協商 Git 傳輸並支援寫入；Dumb HTTP 把 bare repository 當靜態檔案提供，設定簡單但通常只適合唯讀。 | Smart HTTP 可沿用標準 HTTP/HTTPS 入口、同一 URL 和既有 Web 認證；適合接在 Web server 或 reverse proxy 後面。 |

Smart HTTP 把 Git 協定資料放在標準 HTTP(S) 請求中傳送。它可以同時提供公開讀取和需要驗證的寫入，也容易穿過多數只開放 Web 流量的網路。SSH 的憑證管理與主機權限模型較直接；Smart HTTP 則可接現有的 Web 認證。兩者的選擇取決於帳號、授權、網路與維運方式。

Dumb HTTP 沒有 Git client/server 間的智慧協商；伺服器把資料檔案直接提供給 client，並需更新供讀取使用的伺服器資訊。它和可讀寫的 Smart HTTP 是不同服務模式，不能把「repository 放在 Web root 下」等同於建立了安全的 push 服務。

### 4.6 Smart HTTP 的伺服器分工

Pro Git 的範例用 CGI 執行 Git 隨附的 `git-http-backend`。backend 讀取 HTTP 路徑和 headers，判斷 Git client 要 fetch 還是 push，再負責 Git 協定的資料交換。它不負責替使用者驗證身分；TLS、登入驗證、路徑路由和權限仍由呼叫它的 Web server 或前置代理處理。

因此一次 Smart HTTP push 可以分成兩層來看：

1. HTTP 層確認請求能否到達正確 repository，並依服務政策驗證使用者和寫入權限。
2. Git backend 執行 Git 傳輸協定，讓 `git-receive-pack` 收到更新請求和 objects，最後回傳每個 ref 的結果。

書中的 Apache CGI 設定是用來示範分工的簡化範例；它不涵蓋現代部署所需的 TLS、token/SSO、反向代理、請求限制或租戶隔離。實際設定應依目前的 Git、Web server 與身分驗證方式確認。

### 官方 Git 協定補充

Smart HTTP 的讀取協商會先取得 refs 和 server capabilities，再以 POST 傳送 receive-pack 服務所需資料。對 push 來說，client 會提供想更新的 refs、舊值與新值，以及必要的 packfile；伺服器端的 receive-pack 負責驗證並回覆各 ref 的成功或失敗。packfile 是傳輸封裝，不是另一種 commit 或 ref。

### 來源

- Pro Git 第二版：[4.1 The Protocols](https://git-scm.com/book/en/v2/Git-on-the-Server-The-Protocols)、[4.6 Smart HTTP](https://git-scm.com/book/en/v2/Git-on-the-Server-Smart-HTTP)
- Git 官方手冊：[HTTP protocol](https://git-scm.com/docs/http-protocol#_smart_service_git_receive_pack)、[git-receive-pack](https://git-scm.com/docs/git-receive-pack)、[git-http-backend](https://git-scm.com/docs/git-http-backend)

## 和平台設計的關係

- 新 objects 在 pre-receive 執行時位於 quarantine。hook 可以檢查這次 push 帶來的 commit，但不能把 ref 指向尚在 quarantine 的 object。
- pre-receive 拒絕時沒有 ref 更新，Git 會移除 quarantine objects；因此它適合做整批 push 的政策檢查。
- 確認 Web server 到 Git backend 之間的執行身分與 repository 路徑。git-http-backend 會把 CGI 環境提供給 hooks；權限設錯會同時影響讀取、寫入與 hook 執行。

## 延伸閱讀

- [上一章：Git 物件模型](01-git-object-model.md)
- [下一章：伺服器端 hooks 與 reference transaction](03-server-hooks.md)
