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

## 推薦資料

### Pro Git：8.3 Git Hooks

- **連結**：[章節](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)
- **建議閱讀**：Server-Side Hooks，尤其 pre-receive、update、post-receive 的用途差異。
- **免費／語言**：免費線上；繁中網站有導覽，但此節正文主要是英文。
- **預估時間**：20–25 分鐘。
- **讀完應懂**：能分辨 push 前的政策檢查、逐 ref 更新檢查與更新後通知。

### Git 官方 githooks 與 receive-pack

- **連結**：[githooks](https://git-scm.com/docs/githooks)、[git-receive-pack 的 PRE-RECEIVE HOOK 和 QUARANTINE ENVIRONMENT](https://git-scm.com/docs/git-receive-pack#_quarantine_environment)
- **建議閱讀**：pre-receive、reference-transaction、receive-pack 的 pre-receive 與 quarantine 區段。
- **免費／語言**：免費；英文；繁中版未確認。
- **預估時間**：25–35 分鐘。
- **讀完應懂**：能依 hook 的輸入和執行時機判斷它是否適合作為拒絕點或稽核通知點。

## 對 audit log 的設計提醒

pre-receive 可以拒絕 Git refs 更新；reference-transaction 的 prepared 可參與決定 Git ref transaction 是否繼續，而 committed 能通知 refs 已更新。但 Git repository 與外部 SQL 或 event store 不會因此自動成為同一個原子交易。若 hook 在 Git 提交後、外部紀錄前崩潰，兩邊可能不一致。設計時需考慮重試、冪等鍵、重複事件與復原。

## 第一優先自我檢查

1. blob 和 tree 各自保存什麼？為什麼檔名不在 blob 裡？
2. pre-receive 拒絕時，refs 與 quarantine objects 各會怎樣？
3. reference-transaction 在 prepared 回傳非零，和在 committed 階段失敗有何不同？哪個版本才有 preparing？

## 延伸閱讀

- [上一章：git push 與 Smart HTTP](02-smart-http-push.md)
- [下一章：寫到一半當機、WAL 與 journal](04-crash-recovery-wal.md)
