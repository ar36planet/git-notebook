---
title: 寫到一半當機、WAL 與 journal
description: 理解部分寫入、原子性、復原日誌與 checkpoint。
tags:
  - distributed-systems
  - crash-recovery
  - wal
date: 2026-10-05
---

# 寫到一半當機、WAL 與 journal

## 為什麼「寫到一半當機」難處理

一次更新可能橫跨多個檔案、目錄項目、資料庫頁面與記錄。程序或機器可以在任意兩個寫入之間停止；作業系統、檔案系統與儲存裝置也可能重新排序或延遲寫入。重啟後只看到部分變更時，系統需要判斷哪些已提交、哪些要撤銷。

- **undo journal** 通常先記錄舊資料，再覆寫原資料；失效後可依日誌還原。
- **write-ahead log（WAL）** 先把新變更追加到日誌；有 commit record 後，系統可視為已提交，再於 checkpoint 時把日誌內容整併到資料頁。
- 日誌本身也要有可靠的寫入順序、commit 邊界與復原規則；寫入 log 不會自動解決所有耐久性問題。

## 推薦資料

### DDIA 第二版，第 8 章 Transactions

- **連結**：[繁中線上章節](https://ddia.vonng.com/tw/ch8/)，[O’Reilly 英文章節](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781098119058/ch08.html)
- **建議閱讀**：章節開頭和 ACID 的 atomicity、durability；特別看「永續性」段落。併發隔離部分可略讀。
- **免費／語言**：繁中譯文免費線上閱讀；英文原書完整內容需購買／訂閱，預覽免費。
- **預估時間**：25–35 分鐘。
- **讀完應懂**：能分別說明原子性如何處理部分完成，以及永續性如何處理提交後的資料保存。
- **第一版章節對照**：你提供的第 7 章 Transactions 是[這一頁](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch07.html)；目前翻譯站提供的第二版對應第 8 章。

### SQLite 官方 Atomic Commit 與 WAL 文件

- **連結**：[Atomic Commit](https://www.sqlite.org/atomiccommit.html)、[Write-Ahead Logging](https://www.sqlite.org/wal.html)
- **建議閱讀**：Atomic Commit 的 flushing changes to mass storage；WAL 的 How WAL Works 和 Checkpointing。
- **免費／語言**：免費；英文；官方繁中版未確認。
- **預估時間**：25–35 分鐘。
- **讀完應懂**：能用日誌、commit record、recovery 和 checkpoint 解釋資料庫如何從中途失效中恢復。

## 判讀時可問的問題

- 什麼事件代表「已提交」？該事件發生前是否已把必要資料或日誌推到穩定儲存？
- 當機後若不知道最後一筆資料是否寫完，如何用記錄格式或 checksum 判斷？
- 復原流程是重做（redo）、撤銷（undo），還是兩者都有？

## 延伸閱讀

- [[03-server-hooks|上一章：伺服器端 hooks]]
- [[05-clocks-and-pauses|下一章：不可靠的時鐘與程序暫停]]
- [[07-fsync-and-cloud-durability|fsync 與儲存耐久性]]
