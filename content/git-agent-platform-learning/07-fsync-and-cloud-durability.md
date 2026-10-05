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

## 推薦資料

### Dan Luu：Files are hard

- **連結**：[文章](https://danluu.com/file-consistency/)
- **建議閱讀**：Crash Consistency 的 undo log、fsync 與 parent-directory 範例；filesystem semantics 可略讀。
- **免費／語言**：免費；英文；繁中版未確認。
- **預估時間**：25–35 分鐘。
- **讀完應懂**：能理解 syscall 順序、檔案系統與硬體快取如何改變 crash 後的結果。

### LWN：Ensuring data reaches disk

- **連結**：[文章](https://lwn.net/Articles/457667/)
- **建議閱讀**：I/O buffering、fflush、fsync、裝置 write cache。
- **免費／語言**：免費可讀；英文；繁中版未確認。
- **預估時間**：20–25 分鐘。
- **讀完應懂**：能畫出 app、library、kernel page cache、device cache 到 stable storage 的資料路徑。

### Linux man-pages：fsync(2)

- **連結**：[fsync(2)](https://man7.org/linux/man-pages/man2/fsync.2.html)
- **建議閱讀**：DESCRIPTION 中 fsync(file)、metadata 與 directory entry 的說明。
- **免費／語言**：免費；英文；繁中版未確認。
- **預估時間**：10 分鐘。
- **讀完應懂**：能分辨 fsync(file) 與 fsync(directory) 的保證範圍。

## Google Cloud Persistent Disk 耐久性

官方 [Persistent Disk Durability 文件](https://docs.cloud.google.com/compute/docs/disks/persistent-disks#durability_of_persistent_disk)列出的設計耐久性如下。官方頁面可加 **?hl=zh-tw** 選繁體中文。

| Disk type | 官方列出的設計耐久性 |
|---|---:|
| Zonal standard | 高於 99.99% |
| Zonal balanced | 高於 99.999% |
| Zonal SSD | 高於 99.999% |
| Zonal extreme | 高於 99.9999% |
| Regional standard | 高於 99.999% |
| Regional balanced | 高於 99.9999% |
| Regional SSD | 高於 99.9999% |

官方把耐久性定義為在一組硬體故障、災難事件與工程流程假設下，典型磁碟每年的資料遺失機率；表格數字是 disk type 的 aggregate design estimate，**不是有財務賠償的 SLA**。Regional disk 在同一 region 的兩個 zones 間保有 replicas，可協助 zone 故障時維持可用性；它不取代 backup，也不涵蓋客戶誤刪。

### GCP 文件建議閱讀

- **建議閱讀**：Persistent Disk 的 Durability of Persistent Disk 區段，以及 zonal/regional disk 的差異。
- **免費／語言**：官方文件免費；繁體中文頁可用 **?hl=zh-tw**。
- **預估時間**：10–15 分鐘。
- **讀完應懂**：能區分單碟耐久性設計值、服務可用性、SLA 與備份。

## 延伸閱讀

- [[04-crash-recovery-wal|WAL 與 crash recovery]]
- [[06-distributed-locks-fencing|分散式鎖與 fencing token]]
- [[08-linux-systemd-gcp|Linux、systemd 與 GCP 入門]]
