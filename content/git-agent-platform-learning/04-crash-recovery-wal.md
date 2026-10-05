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

## 來源整理：交易、rollback journal 與 WAL

### DDIA：Atomicity 與 Durability

交易把多個資料變更包成一個有明確邊界的操作。**Atomicity（原子性）**回答「部分完成時如何讓狀態看起來像全做或全沒做」；**Durability（耐久性）**回答「系統回報提交成功後，故障重啟是否仍保有結果」。兩者相關但不同：復原日誌能協助原子性，耐久性還要求提交所依賴的資料或日誌確實到達穩定儲存。

DDIA 第二版在第 8 章討論交易與 ACID；第一版同主題在第 7 章。這裡只整理本章需要的原子性和耐久性概念，不延伸整理併發隔離。

### SQLite rollback journal：先保留舊值，再覆寫資料庫

SQLite 的 rollback 模式先把即將修改的舊資料頁寫入 journal。journal 確認落到非揮發性儲存後，SQLite 才能覆寫資料庫頁面；資料庫變更也完成 flush 後，刪除或使 journal 失效才代表提交完成。若在提交完成前當機，下一次開啟資料庫時會把 journal 中的舊頁面寫回去，讓資料庫回到交易開始前的狀態。

此流程的關鍵不是「每次 write 都是原子寫入」，而是讓復原所需的舊資料先可靠保存，並把 journal 的存在狀態當成提交邊界。它仰賴作業系統和儲存裝置正確執行 flush/fsync；如果底層錯誤回報完成，SQLite 無法只靠 journal 修復所有損壞。

### SQLite WAL：追加新值，之後再 checkpoint

WAL 把寫入方向反過來：原資料庫保留舊頁面，修改後的頁面追加到 WAL。交易的 commit record 寫入 WAL 後，交易才算提交。讀取者可依開始讀取時看到的最後一筆 commit record（end mark）取得一致快照；寫入者同時把新頁面追加進 WAL，所以讀取和寫入可以並行，但同一時間仍只有一個 WAL writer。

**Checkpoint** 把 WAL 中已提交的頁面併回主資料庫。仍在讀取舊快照的長交易可能讓 checkpoint 暫停，WAL 因此會累積；要同時考慮讀寫延遲和 checkpoint 時機。SQLite WAL 依賴同一台主機上的共享記憶體索引，不適合放在網路檔案系統供多台主機共同使用。

Rollback journal 和 WAL 是 SQLite 的兩種具體實作。把它們當成原子提交、復原和 checkpoint 的例子即可，不要直接假設其他資料庫使用完全相同的磁碟格式或步驟。

### 來源

- DDIA：[第二版第 8 章 Transactions（繁中）](https://ddia.vonng.com/tw/ch8/)、[第二版英文版](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781098119058/ch08.html)、[第一版第 7 章](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch07.html)
- SQLite 官方文件：[Atomic Commit](https://www.sqlite.org/atomiccommit.html)、[Write-Ahead Logging](https://www.sqlite.org/wal.html)

## 判讀時可問的問題

- 什麼事件代表「已提交」？該事件發生前是否已把必要資料或日誌推到穩定儲存？
- 當機後若不知道最後一筆資料是否寫完，如何用記錄格式或 checksum 判斷？
- 復原流程是重做（redo）、撤銷（undo），還是兩者都有？

## 延伸閱讀

- [上一章：伺服器端 hooks](03-server-hooks.md)
- [下一章：不可靠的時鐘與程序暫停](05-clocks-and-pauses.md)
- [fsync 與儲存耐久性](07-fsync-and-cloud-durability.md)
