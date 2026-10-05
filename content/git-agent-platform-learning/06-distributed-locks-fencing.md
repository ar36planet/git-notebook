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

## 來源整理：租約失效與 fencing

DDIA 第二版第 9 章和 Martin Kleppmann 的文章都強調：分散式鎖服務只能告訴 client「你目前拿到鎖」，不能保證 client 後續每一刻都還在有效期內。client 可能停頓，鎖服務也可能因網路延遲無法及時回應；到期後第二個 client 可以取得鎖，但第一個 client 並不會因此停止執行。

柵欄 token 把「新舊持有者」交由真正保存資料的資源端判斷：鎖服務每次發鎖時發出遞增 token；資源端記住已接受的最大 token，拒絕任何較小值。即使舊程序醒來、誤以為 lease 還有效，它的舊 token 也過不了資源端檢查。token 必須嚴格遞增且不因服務重啟而重複，寫入端也必須在每次修改時驗證；只在 client 裡檢查 lease 時間沒有同等保護力。

這個機制能保護同一資源上的寫入順序，但不會自動讓 Git ref 更新和 SQL 稽核紀錄成為同一筆交易。跨儲存系統仍需要明確的重試、冪等和修復流程。

### 來源

- DDIA：[第二版第 9 章（繁中）](https://ddia.vonng.com/tw/ch9/)、[第二版英文版](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781098119058/ch09.html)
- Martin Kleppmann：[How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)

## 對自架 Git 平台的設計問題

- 寫入 Git refs 的資源端是誰？若有多個 API worker，token 由哪個一致性機制產生？
- 每筆寫入紀錄和 ref 更新如何關聯？接收相同 push 重試時，如何辨認重複事件？
- 如果程序在 Git refs 更新後、稽核資料庫寫入前崩潰，是否有可重播的事件或 reconciliation 流程？

## 第二優先自我檢查

1. 為什麼多步驟寫入可能在當機後只完成一部分？WAL／journal 如何提供復原依據？
2. 為什麼 wall-clock 調整與程序暫停會讓有期限的 lease 失去安全性？
3. fencing token 由誰遞增產生、由誰檢查？新持有者 token 34 已寫入後，儲存服務應如何處理舊 token 33？

## 延伸閱讀

- [上一章：時鐘與程序暫停](05-clocks-and-pauses.md)
- [下一章：fsync 與 Cloud Persistent Disk](07-fsync-and-cloud-durability.md)
