---
title: 分散式鎖、lease 與 fencing token
description: 理解 lock expiry 為何無法停止舊持有者，以及資源端如何用 fencing token 擋住過時寫入。
tags:
  - distributed-systems
  - distributed-locking
  - fencing-token
date: 2026-10-05
---

# 分散式鎖、lease 與 fencing token

## 到期時間不會停止程序

Lease 是有期限的 lock。期限讓其他 client 能在舊持有者消失後接手，但不能讓舊程序在期限到達時自動停止。舊程序可能暫停，lease 到期後新持有者已開始寫入；舊程序恢復後，仍以為自己持有 lock。

## Fencing token 怎麼防止舊寫入

鎖服務每次授予 lock 時都產生嚴格遞增的 token。每次寫入都帶上 token；儲存服務記錄已接受的最大 token，並拒絕比它小的 token。

例如：client A 取得 token 33 後暫停；lease 到期，client B 取得 34 並寫入；A 恢復後帶 33 寫入。儲存端看到 33 小於 34，就拒絕 A。安全性來自資源端檢查 token，不是 client 自己判斷租約是否到期。

## 推薦資料

### DDIA 第二版，第 9 章

- **連結**：[繁中線上章節](https://ddia.vonng.com/tw/ch9/)、[O’Reilly 英文章節](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781098119058/ch09.html)
- **建議閱讀**：「分散式鎖和租約」及「用柵欄機制隔離殭屍與延遲請求」。
- **免費／語言**：繁中譯文免費線上閱讀；英文原書完整內容需購買／訂閱，預覽免費。
- **預估時間**：20–30 分鐘。
- **讀完應懂**：能以失效時間線解釋 lease 競態，並說明 token 必須由儲存端驗證。

### Martin Kleppmann：How to do distributed locking

- **連結**：[文章](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- **建議閱讀**：“The problem with locks” 和 “Making the lock safe with fencing”。
- **免費／語言**：免費；英文；繁中版未確認。
- **預估時間**：20–25 分鐘。
- **讀完應懂**：能說明 process pause、網路延遲與時鐘跳變如何讓 lease 失效，以及 fencing 如何由資源端防護。

## 對自架 Git 平台的設計問題

- 寫入 Git refs 的資源端是誰？若有多個 API worker，token 由哪個一致性機制產生？
- 每筆寫入紀錄和 ref 更新如何關聯？接收相同 push 重試時，如何辨認重複事件？
- 如果程序在 Git refs 更新後、稽核資料庫寫入前崩潰，是否有可重播的事件或 reconciliation 流程？

## 第二優先自我檢查

1. 為什麼多步驟寫入可能在當機後只完成一部分？WAL／journal 如何提供復原依據？
2. 為什麼 wall-clock 調整與程序暫停會讓有期限的 lease 失去安全性？
3. fencing token 由誰遞增產生、由誰檢查？新持有者 token 34 已寫入後，儲存服務應如何處理舊 token 33？

## 延伸閱讀

- [[05-clocks-and-pauses|上一章：時鐘與程序暫停]]
- [[07-fsync-and-cloud-durability|下一章：fsync 與 Cloud Persistent Disk]]
