---
title: 自架 Git 交付平台：繁中筆記
description: 整理 Git 內部、push 與 hooks、故障復原、並行控制、資料耐久性及 Linux 維運的繁中筆記。
tags:
  - git
  - distributed-systems
  - linux
date: 2026-10-05
---

# 自架 Git 交付平台：繁中筆記

這組筆記是給熟悉 C#、ASP.NET 和日常 Git 操作的 .NET 後端工程師。情境是 AI agent 透過 `git push` 交付程式碼，由伺服器接收、檢查、記錄並更新 refs。目標是理解設計取捨，不要求讀完整本書或自行實作 Git 協定。

頁面已把主要概念整理在本文中，可以直接照章節閱讀。外部連結用來標示來源、查核版本或延伸案例，不是必須先去完成的閱讀作業。Pro Git 的相關章節依英文第二版整理成繁體中文摘要；不是官方繁中版，也不是逐段全譯。

## 筆記導覽

| 順序 | 筆記 | 讀完後能回答的問題 |
| --- | --- | --- |
| 1 | [Git 物件模型：commit、tree、blob、ref](01-git-object-model.md) | Git 如何表示快照和歷史？branch、HEAD、tag 分別指向什麼？ |
| 2 | [git push、Smart HTTP 與 receive-pack](02-smart-http-push.md) | push 經過哪些 HTTP 請求？Web server 和 Git backend 各負責什麼？ |
| 3 | [伺服器端 hooks 與 reference transaction](03-server-hooks.md) | 哪個 hook 能拒絕整批 push、拒絕單一 ref，或在提交後通知？ |
| 4 | [寫到一半當機、WAL 與 journal](04-crash-recovery-wal.md) | 交易中斷時如何回復？commit record 和 checkpoint 各代表什麼？ |
| 5 | [不可靠的時鐘與程序暫停](05-clocks-and-pauses.md) | wall-clock、monotonic clock 和跨主機時間戳能回答什麼？ |
| 6 | [分散式鎖、lease 與 fencing token](06-distributed-locks-fencing.md) | 為什麼舊持有者可能在 lease 到期後繼續寫入？資源端如何拒絕它？ |
| 7 | [fsync、目錄耐久性與 Cloud Persistent Disk](07-fsync-and-cloud-durability.md) | write、fsync、目錄同步、磁碟耐久性和備份有何差異？ |
| 8 | [Linux、systemd、cgroup v2 與 GCP](08-linux-systemd-gcp.md) | 如何把服務程序、資源限制、VM、SSH 和防火牆串起來？ |

建議先讀第 1–3 章建立 Git 收件流程，再讀第 4–7 章理解交易、故障和並行問題；第 8 章補足執行環境與維運背景。每章末的問題可用來自我檢查。

## 來源和版本

- 查核日期：2026-10-05。雲端價格、免費額度、服務規格與 Git 文件會變動；部署或估算成本時，以官方當下內容為準。
- Pro Git 第二版由 Scott Chacon 與 Ben Straub 撰寫，採 [CC BY-NC-SA 3.0](https://creativecommons.org/licenses/by-nc-sa/3.0/)。本站 Pro Git 章節為繁中整理式摘要，保留各章英文原文連結並標示來源；分享改作時請遵循相同授權條件。
- DDIA 使用第二版章節編號：Transactions 是第 8 章，The Trouble with Distributed Systems 是第 9 章。第一版相同主題分別在第 7、8 章；連結列於相關筆記。
- reference-transaction 在 Git 2.28 已有 `prepared`、`committed`、`aborted` 階段；`preparing` 是 Git 2.54 加入。細節見[伺服器端 hooks](03-server-hooks.md)。
- GCP Free Tier 和 Persistent Disk 數字具有時效性；相關筆記已標出查核日期。

## Quartz 使用方式

這些頁面是一般 Markdown，使用 Quartz 常見的 title、description、tags、date frontmatter 和標準相對連結，可直接放進 Quartz 的 `content` 目錄；在 GitHub 上也能閱讀和導覽。
