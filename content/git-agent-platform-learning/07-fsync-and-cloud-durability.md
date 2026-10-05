---
title: fsync、目錄耐久性與 Cloud Persistent Disk
description: 區分程式寫入、作業系統快取與穩定儲存，並閱讀 GCP Persistent Disk 耐久性數據。
tags:
  - storage
  - fsync
  - durability
  - gcp
date: 2026-10-05
---

# fsync、目錄耐久性與 Cloud Persistent Disk

## 為什麼 write 成功不代表資料已落盤

資料可能先留在應用程式或 library buffer，再進入 kernel page cache，接著經過檔案系統、儲存裝置的 volatile cache，最後才到穩定儲存。一般 write 返回通常只代表下層接受資料，不一定代表斷電後仍能取回。

Linux 的 fsync(file descriptor) 會要求檔案資料及相關 metadata 同步到儲存裝置。建立新檔案或 rename 後，單獨 fsync 檔案不一定會讓 parent directory 的 directory entry 落盤；若要保證檔名或 rename 的耐久性，還要對目錄 descriptor 做 fsync。每次呼叫的錯誤也必須處理。

即使使用 fsync，仍依賴檔案系統、device cache 與硬體正確回報完成。資料庫會用 journal/WAL 和復原機制，備份則處理更大範圍的資料遺失或誤刪。

## 來源整理：檔案系統的 crash consistency

Dan Luu 的案例和 LWN 對 I/O 路徑的說明，可以合併成一條檢查線：應用程式先把資料交給 library/runtime，再交給 kernel；kernel page cache、檔案系統和裝置快取還可能延遲或重排寫入。程序看到 `write` 成功，通常只代表資料被某一層接受，不能直接推論斷電後內容已安全保存。

`fsync(file)` 要求檔案資料和必要 metadata 同步到儲存裝置。它仍受作業系統、檔案系統和硬體是否正確實作及回報的限制。新建檔案或 rename 後，檔案內容本身落盤，不代表 parent directory 中的檔名或目錄項也已落盤；需要保證目錄項耐久時，還要同步目錄。`rename` 在同一檔案系統中通常提供名稱切換的原子可見性，但原子可見性與斷電後耐久性是兩件事。

crash consistency 的核心問題是「操作被切斷後可能留下哪些組合」。例如先寫暫存檔、同步檔案、rename 到正式路徑、再同步目錄，與直接覆寫正式檔案的故障結果不同。journal/WAL 能記錄復原依據，但它們也必須依正確順序同步；省略錯誤處理或把成功回傳當成永久落盤，仍可能留下資料遺失窗口。

### 來源

- Dan Luu：[Files are hard](https://danluu.com/file-consistency/)
- LWN：[Ensuring data reaches disk](https://lwn.net/Articles/457667/)
- Linux man-pages：[fsync(2)](https://man7.org/linux/man-pages/man2/fsync.2.html)

## Google Cloud Persistent Disk 耐久性

官方 [Persistent Disk Durability 文件](https://docs.cloud.google.com/compute/docs/disks/persistent-disks#durability_of_persistent_disk)列出的設計耐久性如下。官方頁面可加 **?hl=zh-tw** 選繁體中文。

| Disk type         | 官方列出的設計耐久性 |
| ----------------- | -------------------: |
| Zonal standard    |          高於 99.99% |
| Zonal balanced    |         高於 99.999% |
| Zonal SSD         |         高於 99.999% |
| Zonal extreme     |        高於 99.9999% |
| Regional standard |         高於 99.999% |
| Regional balanced |        高於 99.9999% |
| Regional SSD      |        高於 99.9999% |

官方把耐久性定義為在一組硬體故障、災難事件與工程流程假設下，典型磁碟每年的資料遺失機率；表格數字是 disk type 的 aggregate design estimate，**不是有財務賠償的 SLA**。Regional disk 在同一 region 的兩個 zones 間保有 replicas，可協助 zone 故障時維持可用性；它不取代 backup，也不涵蓋客戶誤刪。

### GCP 數據解讀

官方文件提供不同 Persistent Disk 類型的設計耐久性估值，並區分單一 zone 磁碟與跨兩個 zones 複寫的 regional disk。這些值描述的是在官方假設下的資料遺失機率估計，不等於服務可用性保證或有賠償條款的 SLA。Regional disk 可降低單一 zone 故障造成的中斷風險，但仍需備份以處理誤刪、錯誤覆寫或更大範圍的事故。數值可能更新，建立資源前請以官方頁面為準。

## 延伸閱讀

- [WAL 與 crash recovery](04-crash-recovery-wal.md)
- [分散式鎖與 fencing token](06-distributed-locks-fencing.md)
- [Linux、systemd 與 GCP 入門](08-linux-systemd-gcp.md)
