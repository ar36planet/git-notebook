---
title: Git 物件模型：commit、tree、blob、ref
description: 用物件圖理解 Git 的快照、提交歷史、分支與參照。
tags:
  - git
  - git-internals
date: 2026-10-05
---

# Git 物件模型：commit、tree、blob、ref

## 先建立的心智模型

- **blob** 保存檔案內容，不保存檔名。
- **tree** 保存目錄項目，包括名稱、檔案模式，以及指向 blob 或子 tree 的 object ID。
- **commit** 指向一個 tree，並記錄 parent commit、作者、提交者與訊息。合併 commit 可以有多個 parent。
- **ref** 是方便閱讀的名稱，保存某個 object ID。branch ref 通常指向 commit，例如 **refs/heads/main**。
- commit、tree、blob 是以內容定址的物件。建立 branch 主要是建立一個 ref；建立 commit 時 Git 產生新物件，再移動 branch ref。
- **HEAD** 通常是 symbolic ref，指向目前 branch。Annotated tag ref 可以指向 tag object，所以不是所有 ref 都直接指向 commit。

可以把提交歷史想成一張圖：commit 透過 parent 形成歷史；commit 指向 tree；tree 再往下指向 blob 和子 tree。branch 名稱只標示圖上的某個 commit。

## 推薦資料

### Pro Git：Git Internals 10.1–10.3

- **連結**：[繁中書目與章節索引](https://git-scm.com/book/zh-tw/v2)、[10.1 Plumbing and Porcelain](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain)、[10.2 Git Objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)、[10.3 Git References](https://git-scm.com/book/en/v2/Git-Internals-Git-References)
- **建議閱讀**：10.1 讀 object database 和 refs 概觀；10.2 讀 Git Objects 與 Tree Objects；10.3 讀 References、Branches 與 HEAD。
- **免費／語言**：免費線上，Pro Git 第二版（2014），CC BY-NC-SA 3.0。網站將繁中列為部分翻譯；以上章節正文主要是英文。
- **預估時間**：30–40 分鐘。
- **讀完應懂**：能從 commit、tree、blob、ref 解釋一次 commit 如何描述檔案快照，以及 branch 為何只是可移動的 ref。

### Mary Rose Cook：Git from the inside out

- **連結**：[文章](https://maryrosecook.com/blog/post/git-from-the-inside-out)；頁面也連到演講影片。
- **建議閱讀**：從建立 repository、git add、commit 到移動 branch 的步驟；後段實作可略讀。
- **免費／語言**：免費，英文；繁中版未確認。範例使用 SHA-1 表示 object ID，作為圖解即可。
- **預估時間**：20–25 分鐘。
- **讀完應懂**：能用圖形理解 object graph 如何解釋 Git 的提交與分支行為。

### Object ID 的實作注意事項

Git 預設使用 SHA-1，也支援 SHA-256 repository。官方 [git-init 文件](https://git-scm.com/docs/git-init)列出 sha1 與 sha256 兩種 object format，並註明目前兩種格式的 repository 尚不能互通。若平台要保存 object ID，請視為不透明值，不要把 40 字元當成永久規格。

## 延伸閱讀

- [下一章：git push、Smart HTTP 與 receive-pack](02-smart-http-push.md)
- [伺服器端 hooks 與 reference transaction](03-server-hooks.md)
