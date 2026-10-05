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

## 來源整理：DDIA 的時鐘與 Linux clock API

DDIA 第二版第 9 章指出，日曆時鐘會受時鐘同步和人工調整影響；主機之間的時鐘偏差與網路延遲，使時間戳無法單獨證明先後順序。單調時鐘適合比較同一主機上的經過時間，但不同主機各自的單調時鐘沒有可直接比較的共同起點。程序暫停則會讓原本看似安全的時間判斷失效：lease 可能已過期，但暫停中的舊持有者醒來後仍繼續執行。

Linux `clock_gettime(2)` 將這些用途反映成不同 clock：

- `CLOCK_REALTIME` 是可設定的日曆時間，適合記錄日期時間；校時可能使它跳動。
- `CLOCK_MONOTONIC` 是單調的經過時間基準，不受 `CLOCK_REALTIME` 的人工跳時影響，但不計入系統 suspend 的時間。
- `CLOCK_BOOTTIME` 具有 monotonic 特性，且會把系統 suspend 的時間算入經過時間。

實務上，量測逾時通常選 monotonic 類時鐘；寫 log 時使用 wall-clock，並保留主機或時鐘來源資訊。若要判斷跨主機事件因果，需使用協定、序號或因果資訊，不能只排序 timestamp。

### 來源

- DDIA：[第二版第 9 章 The Trouble with Distributed Systems（繁中）](https://ddia.vonng.com/tw/ch9/)、[第二版英文版](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781098119058/ch09.html)、[第一版第 8 章](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch08.html)
- Linux man-pages：[clock_gettime(2)](https://man7.org/linux/man-pages/man2/clock_gettime.2.html)

## 延伸閱讀

- [上一章：Crash recovery 與 WAL](04-crash-recovery-wal.md)
- [下一章：分散式鎖、lease 與 fencing token](06-distributed-locks-fencing.md)
