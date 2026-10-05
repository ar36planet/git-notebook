---
title: Linux、systemd、cgroup v2 與 GCP
description: 補足 Linux 命令列、服務程序管理、cgroup 與 Compute Engine 基礎。
tags:
  - linux
  - systemd
  - cgroup
  - gcp
date: 2026-10-05
---

# Linux、systemd、cgroup v2 與 GCP

本章把前面討論的 Git backend 放回實際執行環境：shell 負責啟動和串接命令，systemd 管理服務生命週期，cgroup 限制程序資源，Compute Engine 提供 VM、網路和磁碟。重點是讀懂各層責任，不是照範例直接部署。

## Linux shell 和程序

檔案路徑分成絕對路徑與相對路徑；常用操作包括列出目錄、建立或移動檔案、檢查擁有者和讀寫執行權限，以及查看目前程序。服務故障排查常從「執行了哪個命令、使用哪個帳號、檔案權限如何、程序是否還在」開始。

Shell 透過標準輸入、標準輸出和標準錯誤串接程式。pipe 把前一個程序的輸出接到下一個程序的輸入；redirect 把輸入或輸出接到檔案；exit status 則讓呼叫端判斷命令成功或失敗。這些規則也影響 hook 或啟動腳本如何傳遞錯誤。

**來源：** William Shotts 的 [The Linux Command Line](https://linuxcommand.org/tlcl.php)；MIT [Missing Semester：Shell](https://missing.csail.mit.edu/2026/course-shell/)。

## systemd：服務啟動和停止

systemd service unit 描述如何啟動、停止和追蹤一個服務。`ExecStart` 指定啟動命令；`ExecStop` 可指定停止流程；`KillSignal` 決定停止時先送出的訊號；`TimeoutStopSec` 設定等待程序結束的時間。`KillMode=control-group` 會把 unit 的 control group 作為停止範圍，因此由服務啟動的子程序也納入管理。部署時要讓程式能接收終止訊號並完成清理，逾時後再依設定強制結束。

**來源：** systemd 官方 [`systemd.service`](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html) 和 [`systemd.kill`](https://www.freedesktop.org/software/systemd/man/latest/systemd.kill.html) 手冊。

## cgroup v2：追蹤和限制資源

cgroup v2 以統一階層組織程序，控制器可對一組程序做資源計量或限制。CPU controller 管理 CPU 使用分配，memory controller 管理記憶體用量和限制。systemd 通常會替 unit 建立 cgroup，因此服務的程序樹和資源管理可以跟 unit 一起操作。cgroup 負責限制與統計資源，不負責應用程式資料的持久化。

**來源：** Linux kernel 官方 [Control Group v2 文件](https://docs.kernel.org/admin-guide/cgroup-v2.html)。

## GCP Compute Engine：VM、SSH 和網路入口

建立 VM 時要選 region/zone、machine type 和 boot disk；SSH-in-browser 或 `gcloud` SSH 用來管理主機。VPC firewall rule 會依 target、source、protocol 和 port 決定哪些流量可進入 VM。應分開看管理連線與應用程式對外服務的 ingress 規則，避免為了方便而讓服務埠對所有來源開放。

**來源：** Google Cloud 官方繁中頁面：[建立 Linux VM](https://docs.cloud.google.com/compute/docs/create-linux-vm-instance?hl=zh-tw)、[SSH 連線](https://docs.cloud.google.com/compute/docs/instances/ssh?hl=zh-tw)、[VPC firewall rules](https://docs.cloud.google.com/firewall/docs/firewalls?hl=zh-tw)。

## Free Tier 與預算

Free Tier 是按月計算的特定產品額度；Free Trial 是新客戶在限期內可用的抵用額，兩者不是同一種免費承諾。額度可能只適用指定 VM 規格、區域、磁碟和流量；超出範圍或用量可能收費。

截至 2026-10-05，官方列出每月合計 1 台非 preemptible e2-micro 的月時數，限 us-west1、us-central1、us-east1；另有 30 GB-month standard persistent disk，以及每月 1 GB 從北美傳出的資料（中國與澳洲除外）。這不是台灣 region 的免費 VM。新客戶 US$300／90 天 Free Trial 是另一種抵用額，不代表之後所有資源免費；未升級 billing account 時，試用結束的專案和資源會停止運作。

建立資源前請重新確認 [Google Cloud Free Tier 與試用額度](https://docs.cloud.google.com/free/docs/free-cloud-features?hl=zh-tw)，並檢查磁碟、網路流量、區域和 VM 規格是否符合條件。

## 返回路線

- [學習路線首頁](index.md)
- [上一章：fsync 與 Cloud Persistent Disk](07-fsync-and-cloud-durability.md)
- [第一優先：Git 物件模型](01-git-object-model.md)
