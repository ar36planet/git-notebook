---
title: 伺服器端 hooks 與 reference transaction
description: 比較 pre-receive 與 reference-transaction，理解拒絕 push 和記錄 ref 更新的時機。
tags:
  - git
  - hooks
  - audit-log
date: 2026-10-05
---

# 伺服器端 hooks 與 reference transaction

## pre-receive：push 層級的政策檢查

pre-receive 由 git-receive-pack 在更新 refs 前呼叫，每次 receive 操作一次。stdin 每行包含 **old-value、new-value、ref-name**，代表一個待更新 ref。

- 非零退出碼會拒絕整次 push 的 ref 更新。
- 執行時新 objects 還在 quarantine，hook 可以檢查 incoming commit；若 hook 拒絕，Git 移除 quarantine objects。
- 適合檢查整批 push 的政策，例如允許哪些 refs、commit 是否符合規則、要記錄什麼 metadata。

## reference-transaction：ref 更新交易的生命週期

這個 hook 在任何會更新 refs 的 Git 指令中都可能執行，因此不只與 push 有關。stdin 同樣提供每個 ref 的 old/new/ref-name。

| 狀態      | 發生時機                                               | 可以怎麼理解                               |
| --------- | ------------------------------------------------------ | ------------------------------------------ |
| preparing | Git 2.54 起；更新已排入 transaction，ref lock 尚未取得 | 早期通知，可在正式鎖定前拒絕               |
| prepared  | 更新已排入 transaction，refs 已鎖定                    | 變更已準備好；非零退出可以取消 transaction |
| committed | ref transaction 已提交，refs 已有新值                  | 已提交結果通知；hook 失敗不能回滾 refs     |
| aborted   | transaction 取消，沒有套用變更，鎖已釋放               | 取消結果通知                               |

preparing 和 prepared 階段的非零退出會取消 transaction；committed／aborted 階段的退出狀態不改變結果。

## 重要版本差異

- [Git 2.27 的 githooks 文件](https://git-scm.com/docs/githooks/2.27.0.html)沒有 reference-transaction。
- [Git 2.28 的 githooks 文件](https://git-scm.com/docs/githooks/2.28.0.html)已有 reference-transaction，列出 prepared、committed、aborted。可說此 hook 從 Git 2.28 起可用。
- [Git 2.54 release notes](https://github.com/git/git/blob/master/Documentation/RelNotes/2.54.0.adoc)記載新增 preparing 狀態，發行日期為 2026-04-20。
- [最新版 githooks 手冊](https://git-scm.com/docs/githooks#_reference_transaction)描述四個階段與 stdin 格式。

## Pro Git 第 8.3 節原文整理

以下依 Pro Git 第二版英文第 8.3 節翻譯整理，聚焦本章會用到的伺服器端 hooks；不是逐段直譯。Pro Git 由 Scott Chacon 與 Ben Straub 撰寫，採 CC BY-NC-SA 3.0。

Git hooks 是 Git 在特定操作前後呼叫的可執行程式，分成 client-side 和 server-side。client-side hook 由本機操作觸發，使用者可以略過或自行改動；伺服器要強制執行的規則，必須放在自己控制的伺服器端。

Hooks 放在 repository 的 Git directory 下 `hooks/`。`git init` 會放入範例檔，檔名以 `.sample` 結尾；要啟用範例需移除後綴並確保可執行。hook 可用任何伺服器可執行的程式語言撰寫，輸入資料和退出碼則由各 hook 的介面決定。

Pro Git 將 push 相關伺服器 hooks 分成三種用途：

| Hook | 呼叫次數與輸入 | 拒絕範圍／常見用途 |
| --- | --- | --- |
| `pre-receive` | 一次 push 呼叫一次；stdin 列出此次要求更新的 refs。 | 非零退出會拒絕這次 push 的所有 ref 更新。適合檢查整批更新共同的政策，例如不可 force-push、使用者是否能修改指定 refs。 |
| `update` | 每個待更新的 ref 呼叫一次；參數包含 ref 名稱、舊 object ID、新 object ID。 | 非零退出只拒絕該 ref，其他 ref 仍可能成功。適合逐分支套用規則。 |
| `post-receive` | 至少一個 ref 更新成功後呼叫；stdin 列出成功更新的 refs。 | 已無法拒絕 push。適合通知 CI、更新其他服務或送出事件；若處理很久，client 連線也會等它完成。 |

要選 hook，先問「這個檢查要否決整批 push、單一 ref，還是只在成功後通知？」拒絕點和通知點不能互換：`post-receive` 不能用來回滾已成功的 Git 更新。

## Git 官方手冊補充：reference transaction 與 quarantine

Pro Git 第 8.3 節說明了常見 hooks 的使用方式；目前 Git 官方手冊另外定義 `reference-transaction`，讓 hook 參與 ref transaction 的不同階段。它可能由 push 以外的 ref 更新操作觸發。`prepared` 階段可在 ref transaction 寫入前拒絕；`committed` 是 ref 已更新後的通知，失敗不能讓已提交的 refs 倒退。Git 2.54 加入 `preparing` 階段；Git 2.28 起已有 `reference-transaction`，但當時沒有 `preparing`。

receive-pack 在執行 `pre-receive` 時會把這次 push 傳入的新物件放在 quarantine 區。hook 可以檢查 incoming commits；若整批 push 被拒絕，Git 會清除 quarantine 中的物件。這保護了 object database 免於留下未被 refs 接受的資料，但它不會讓 Git refs 和外部資料庫自動形成同一筆原子交易。

### 來源

- Pro Git 第二版：[8.3 Git Hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)
- Git 官方手冊：[githooks](https://git-scm.com/docs/githooks)、[git-receive-pack quarantine environment](https://git-scm.com/docs/git-receive-pack#_quarantine_environment)、[Git 2.54 release notes](https://github.com/git/git/blob/master/Documentation/RelNotes/2.54.0.adoc)

## 對 audit log 的設計提醒

pre-receive 可以拒絕 Git refs 更新；reference-transaction 的 prepared 可參與決定 Git ref transaction 是否繼續，而 committed 能通知 refs 已更新。但 Git repository 與外部 SQL 或 event store 不會因此自動成為同一個原子交易。若 hook 在 Git 提交後、外部紀錄前崩潰，兩邊可能不一致。設計時需考慮重試、冪等鍵、重複事件與復原。

## 第一優先自我檢查

1. blob 和 tree 各自保存什麼？為什麼檔名不在 blob 裡？
2. pre-receive 拒絕時，refs 與 quarantine objects 各會怎樣？
3. reference-transaction 在 prepared 回傳非零，和在 committed 階段失敗有何不同？哪個版本才有 preparing？

## 延伸閱讀

- [上一章：git push 與 Smart HTTP](02-smart-http-push.md)
- [下一章：寫到一半當機、WAL 與 journal](04-crash-recovery-wal.md)
