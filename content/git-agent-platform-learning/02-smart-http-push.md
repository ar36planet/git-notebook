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

## 推薦資料

### Pro Git：第 4 章，4.1 與 4.6

- **連結**：[繁中書目與章節索引](https://git-scm.com/book/zh-tw/v2)、[4.1 Protocols](https://git-scm.com/book/en/v2/Git-on-the-Server-The-Protocols)、[4.6 Smart HTTP](https://git-scm.com/book/en/v2/Git-on-the-Server-Smart-HTTP)
- **建議閱讀**：4.1 的 HTTP 與 Smart HTTP 介紹；4.6 Smart HTTP 的 CGI 設定概念和驗證範例。不要把舊 Apache 範例直接當成現代部署設定。
- **免費／語言**：免費線上，CC BY-NC-SA 3.0；繁中網站有章節導覽，但相關正文主要為英文。
- **預估時間**：20–30 分鐘。
- **讀完應懂**：能分辨 Smart HTTP 與 Dumb HTTP，以及 Smart HTTP 如何使用標準 HTTP/S 埠和 receive-pack 支援 push。

### Git 官方協定與伺服器手冊

- **連結**：[HTTP protocol 的 Smart Service git-receive-pack](https://git-scm.com/docs/http-protocol#_smart_service_git_receive_pack)、[git-receive-pack](https://git-scm.com/docs/git-receive-pack)、[git-http-backend](https://git-scm.com/docs/git-http-backend)
- **建議閱讀**：HTTP protocol 的 ref discovery 與 git-receive-pack 範例；receive-pack 的 DESCRIPTION、PRE-RECEIVE HOOK、QUARANTINE ENVIRONMENT；http-backend 的 DESCRIPTION、SERVICES。
- **免費／語言**：免費，Git 官方英文手冊；繁中譯本未確認。
- **預估時間**：25–35 分鐘。
- **讀完應懂**：能認出 GET ref discovery、POST receive-pack、packfile、quarantine 和 push status 的位置。

## 和平台設計的關係

- 新 objects 在 pre-receive 執行時位於 quarantine。hook 可以檢查這次 push 帶來的 commit，但不能把 ref 指向尚在 quarantine 的 object。
- pre-receive 拒絕時沒有 ref 更新，Git 會移除 quarantine objects；因此它適合做整批 push 的政策檢查。
- 確認 Web server 到 Git backend 之間的執行身分與 repository 路徑。git-http-backend 會把 CGI 環境提供給 hooks；權限設錯會同時影響讀取、寫入與 hook 執行。

## 延伸閱讀

- [[01-git-object-model|上一章：Git 物件模型]]
- [[03-server-hooks|下一章：伺服器端 hooks 與 reference transaction]]
