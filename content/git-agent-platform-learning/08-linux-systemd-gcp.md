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

這是第四優先的後續材料。先學到能讀懂 unit file、追程序與設定 VM 即可，不必把整本 Linux 命令列書讀完。

## Linux 基本操作

### The Linux Command Line — William Shotts

- **連結**：[作者網站、目錄與第七版免費 PDF](https://linuxcommand.org/tlcl.php)
- **建議閱讀**：Navigation、Exploring the System、Manipulating Files and Directories、Permissions、Processes。
- **免費／語言**：第七版 Internet Edition 英文 PDF 免費下載，CC BY-NC-ND；有繁中紙本《Linux 指令大全》，但查到的台灣版為 2022 年譯自第二版，較目前免費英文版落後。可參考[國家圖書館 ISBN 記錄](https://isbn.ncl.edu.tw/NEW_ISBNNet/main_DisplayRecord_Popup.php?Pact=view&Pkey=1110426%2A0077)。
- **預估時間**：精選章節約 1.5–2 小時。
- **讀完應懂**：能在 Linux shell 找檔案、讀權限、檢視程序並操作基本檔案。

### MIT Missing Semester 2026：Introduction to the Shell

- **連結**：[課程講義與影片](https://missing.csail.mit.edu/2026/course-shell/)
- **建議閱讀**：shell basics、pipes、redirects；可先看講義，不必做完所有練習。
- **免費／語言**：免費；英文；繁中版未確認。
- **預估時間**：30–45 分鐘。
- **讀完應懂**：能理解 shell 用 stdin、stdout、pipe 和 exit status 串接程式。

## systemd 與 cgroup v2

### systemd.service 與 systemd.kill

- **連結**：[systemd.service](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)、[systemd.kill](https://www.freedesktop.org/software/systemd/man/latest/systemd.kill.html)
- **建議閱讀**：systemd.service 的 Service、ExecStart、ExecStop；systemd.kill 的 KillMode、KillSignal、TimeoutStopSec。
- **免費／語言**：systemd 官方文件免費；英文；繁中官方譯本未確認。
- **預估時間**：25–35 分鐘。
- **讀完應懂**：能讀基本 unit file，並知道 KillMode=control-group 會在停止 unit 時處理該 control group 中的程序。預設先送 SIGTERM，逾時後再依設定送 SIGKILL。

### Linux kernel Control Group v2

- **連結**：[官方文件](https://docs.kernel.org/admin-guide/cgroup-v2.html)
- **建議閱讀**：Introduction、Basic Operations、Controllers 的開頭和 CPU／memory 概念。
- **免費／語言**：免費；英文；繁中官方版未確認。
- **預估時間**：20–30 分鐘。
- **讀完應懂**：能把 cgroup v2 看成統一階層式程序集合，可做資源管理與限制。

## GCP Compute Engine

### 建立 Linux VM、SSH 與防火牆

- **連結**：[建立 Linux VM](https://docs.cloud.google.com/compute/docs/create-linux-vm-instance?hl=zh-tw)、[SSH 連線](https://docs.cloud.google.com/compute/docs/instances/ssh?hl=zh-tw)、[VPC firewall rules](https://docs.cloud.google.com/firewall/docs/firewalls?hl=zh-tw)
- **建議閱讀**：建立 VM 的 region/zone、machine type、boot disk；SSH-in-browser 或 gcloud SSH；firewall rule 的 target、source、protocol/port。
- **免費／語言**：Google 官方文件免費，有繁體中文頁面。
- **預估時間**：35–50 分鐘。
- **讀完應懂**：能建立 Linux VM、用 SSH 登入，並把管理連線和對外服務 ingress rule 分開理解。

### Google Cloud Free Tier

- **連結**：[Free Cloud features and trial offer](https://docs.cloud.google.com/free/docs/free-cloud-features?hl=zh-tw)
- **建議閱讀**：Compute Engine Free Tier 與 Free Trial 兩段。
- **免費／語言**：官方繁中說明免費；額度會變動，建立資源前重查。
- **預估時間**：10–15 分鐘。
- **讀完應懂**：能分清按月 Free Tier 與新客戶限期抵用額，並找出超額會計費的項目。

截至 2026-10-05，官方列出每月合計 1 台非 preemptible e2-micro 的月時數，限 us-west1、us-central1、us-east1；另有 30 GB-month standard persistent disk，以及每月 1 GB 從北美傳出的資料（中國與澳洲除外）。這不是台灣 region 的免費 VM。新客戶 US$300／90 天 Free Trial 是另一種抵用額，不代表之後所有資源免費；未升級 billing account 時，試用結束的資源可能被停止或刪除。

## 返回路線

- [學習路線首頁](index.md)
- [上一章：fsync 與 Cloud Persistent Disk](07-fsync-and-cloud-durability.md)
- [第一優先：Git 物件模型](01-git-object-model.md)
