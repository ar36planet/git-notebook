---
title: 自架 Git 交付平台：學習路線
description: 以能判斷設計取捨為目標，整理 Git 內部、故障與並行、資料耐久性及 Linux 維運資料。
tags:
  - git
  - distributed-systems
  - linux
date: 2026-10-05
---

# 自架 Git 交付平台：學習路線

這組筆記是給熟悉 C#、ASP.NET 和日常 Git 操作的 .NET 後端工程師。目標是看懂「AI agent 用 git push 交付程式碼，伺服器記錄並把關每次寫入」所涉及的設計，不要求讀完整本書，也不要求能獨立實作 Git、分散式協定或 Linux 維運。

## 章節

1. [[01-git-object-model|Git 物件模型：commit、tree、blob、ref]]
2. [[02-smart-http-push|git push、Smart HTTP 與 receive-pack]]
3. [[03-server-hooks|伺服器端 hooks 與 reference transaction]]
4. [[04-crash-recovery-wal|寫到一半當機、WAL 與 journal]]
5. [[05-clocks-and-pauses|不可靠的時鐘與程序暫停]]
6. [[06-distributed-locks-fencing|分散式鎖、lease 與 fencing token]]
7. [[07-fsync-and-cloud-durability|fsync、目錄耐久性與 Cloud Persistent Disk]]
8. [[08-linux-systemd-gcp|Linux、systemd、cgroup v2 與 GCP]]

## 查核與版本註記

- 查核日期：2026-10-05。價格、雲端免費額度與 Git 文件會改動，部署前請再看官方頁面。
- Pro Git 第二版線上免費，採 CC BY-NC-SA 3.0。官方網站將繁體中文列為部分翻譯；本系列使用到的 Internals 10.1–10.3、Smart HTTP 4.6 與 Hooks 8.3 頁面，導覽雖有繁中，正文主要仍是英文。
- 你提供的 [Vonng/DDIA 繁中翻譯庫](https://github.com/Vonng/ddia/blob/main/content/tw/_index.md)目前提供第二版譯文。第二版 Transactions 在第 8 章，The Trouble with Distributed Systems 在第 9 章；第一版相同主題分別是第 7、8 章。譯文可免費線上閱讀，但請查看 repo 法律聲明；免費閱讀不等於可任意再散布。
- 若要核對你原先提到的第一版章節編號，O’Reilly 的正確頁面是[第 7 章 Transactions](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch07.html)和[第 8 章 The Trouble with Distributed Systems](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch08.html)。英文版完整內容需購買／訂閱，章節預覽免費。
- reference-transaction hook 在 Git 2.28 已存在；當時只有 prepared、committed、aborted。preparing 是 Git 2.54 新增的階段。細節見 [[03-server-hooks|伺服器端 hooks]]。
- The Linux Command Line 作者網站提供第七版 Internet Edition 免費英文 PDF；查到的繁體中文紙本《Linux 指令大全》是 2022 年譯自第二版，內容較舊。細節見 [[08-linux-systemd-gcp|Linux 與維運]]。
- 「未確認」表示沒有找到足以確認該語言版本或資訊的來源。閱讀時間為專注閱讀估計，不含做筆記。

## 一週閱讀計畫：第一、二優先，每天約一小時

| 日 | 閱讀安排 | 當天目標 |
|---|---|---|
| 第 1 天 | [[01-git-object-model|Git 物件模型]]：Pro Git 10.2（約 25 分）＋ Mary Rose Cook 文章前半（約 30 分） | 畫出 blob、tree、commit 的連結。 |
| 第 2 天 | [[01-git-object-model|Git 物件模型]]：Pro Git 10.1、10.3（約 35 分）＋ SHA-256 註記（約 15 分） | 解釋 branch ref、HEAD 與 commit graph 的不同。 |
| 第 3 天 | [[02-smart-http-push|Push 與 Smart HTTP]]：Pro Git 4.1、4.6（約 30 分）＋ HTTP protocol receive-pack 範例（約 25 分） | 畫出 ref discovery、POST receive-pack、packfile 的順序。 |
| 第 4 天 | [[03-server-hooks|伺服器端 hooks]]：Pro Git 8.3（約 20 分）＋ githooks 與 receive-pack（約 40 分） | 說明 pre-receive 和 reference-transaction 各能把關什麼。 |
| 第 5 天 | [[04-crash-recovery-wal|Crash recovery 與 WAL]]：DDIA 第二版第 8 章選讀（約 30 分）＋ SQLite WAL（約 25 分） | 說明 commit record、復原與 checkpoint。 |
| 第 6 天 | [[05-clocks-and-pauses|時鐘與程序暫停]]：DDIA 第 9 章選讀（約 45 分）＋ clock_gettime(2)（約 10 分） | 分辨 wall-clock、monotonic clock 與跨主機時間戳。 |
| 第 7 天 | [[06-distributed-locks-fencing|分散式鎖與 fencing]]：DDIA 第 9 章選讀（約 30 分）＋ Kleppmann 部落格（約 25 分） | 解釋為什麼資源端要檢查遞增 token。 |

第三、四優先可以等 Git 的 push 與 hook 設計穩定後再讀。每章只列本專案需要的段落；若自我檢查題答不順，重讀對應段落即可。

## Quartz 使用方式

這些頁面是一般 Markdown，已加上 Quartz 常用的 title、description、tags、date frontmatter，並使用 Quartz 支援的 wikilink。之後可將整個資料夾複製到 Quartz 專案的 content 目錄；本資料夾獨立於工作區裡既有的 wiki 筆記。
