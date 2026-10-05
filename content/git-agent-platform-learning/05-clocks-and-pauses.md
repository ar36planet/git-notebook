---
title: 不可靠的時鐘與程序暫停
description: 區分 wall-clock 與 monotonic clock，並理解跨主機時間與程序暫停的風險。
tags:
  - distributed-systems
  - clocks
  - process-pauses
date: 2026-10-05
---

# 不可靠的時鐘與程序暫停

## 時鐘能回答什麼，不能回答什麼

**wall-clock**（日曆／牆上時鐘）適合記錄日期時間，但可能被 NTP、管理者調整或閏秒影響而跳變。**monotonic clock** 適合量測同一台主機上的經過時間，不應倒退；它不能直接告訴你兩台主機上的事件先後。

網路傳遞有不確定延遲。就算兩台主機都同步 NTP，也只能把誤差控制在某個範圍，不會變成同一個完美時鐘。時間戳接近或相同，不足以證明事件有因果順序。

程序可以因 GC、VM 暫停、CPU 飢餓、page fault、SIGSTOP 或主機休眠而長時間不執行。程序醒來時，某個 lease 可能早已過期。

## 推薦資料

### DDIA 第二版，第 9 章 The Trouble with Distributed Systems

- **連結**：[繁中線上章節](https://ddia.vonng.com/tw/ch9/)、[O’Reilly 英文章節](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781098119058/ch09.html)
- **建議閱讀**：「不可靠的時鐘」中的日曆時鐘、單調時鐘、clock synchronization and accuracy，以及「程序暫停」。
- **免費／語言**：繁中譯文免費線上閱讀；英文原書完整內容需購買／訂閱，預覽免費。
- **預估時間**：30–40 分鐘。
- **讀完應懂**：能區分測量 duration 和記錄 point in time，並說明為何跨主機時間戳不保證事件排序。
- **第一版章節對照**：你提供的第 8 章 The Trouble with Distributed Systems 是[這一頁](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch08.html)；目前翻譯站提供的第二版對應第 9 章。

### Linux man-pages：clock_gettime(2)

- **連結**：[clock_gettime(2)](https://man7.org/linux/man-pages/man2/clock_gettime.2.html)
- **建議閱讀**：CLOCK_REALTIME、CLOCK_MONOTONIC、CLOCK_BOOTTIME 的說明。
- **免費／語言**：免費；英文；繁中版未確認。
- **預估時間**：10–15 分鐘。
- **讀完應懂**：能將 DDIA 的時鐘概念對應到 Linux API，並看出 monotonic 只保證單機時間不倒退。

## 延伸閱讀

- [上一章：Crash recovery 與 WAL](04-crash-recovery-wal.md)
- [下一章：分散式鎖、lease 與 fencing token](06-distributed-locks-fencing.md)
