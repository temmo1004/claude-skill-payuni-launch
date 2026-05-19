# Audit 自我加固迴圈 — Dual-AI Self-Audit 心法

**重要訂正**：PAYUNi 是**一次過審**。R1-R9 八輪修復是**自己對自己**跑 dual-AI self-audit 不滿意才加碼修的，不是 PAYUNi 退件。

PAYUNi 標準是「能通過 demo + 政策一致 + DB 對得起來」就會放行。但**過審 ≠ 你的系統真的穩**。實戰經驗：第一輪能過審的版本，dual-AI 還是會找出 90+ 個 race / idempotency / barrier 漏洞 — 不修也能上，修了才不會在上 prod 後爆。

這檔講「過審門檻」vs「我自己想到的標準」差距怎麼補。

## 兩個層次

| 層次 | 標準訂定者 | 內容 | 不做會怎樣 |
|---|---|---|---|
| L1 **PAYUNi 過審** | PAYUNi 審查員 | refund ledger / deletion / policy 一致 / Customer Portal / 範圍收斂 | 退件，不能上線收錢 |
| L2 **我自己滿意** | 你 + dual-AI | webhook retry double-grant / 退費邊界 / TWD 聲明 / race window / promo union check / quota atomic | 上 prod 後 1-3 個月內慢慢爆，客訴、對帳出錯、套利 |

L1 做完就能過審。L2 做完才能晚上睡得著。

## 真實 timeline（reference）

第一次送 PAYUNi 就過審。後續一週內自己跑 dual-AI 自審：

| 階段 | 觸發 | 內容 | finding 數 |
|---|---|---|---|
| L1 完成 | PAYUNi 送審 → 過 | refund ledger / deletion / Portal / policy 一致 | — |
| **L2 R4** | 自審：deletion 是否真的沒 race？ | 22 個 race window（R4B01-R4B07） | ~22 |
| **L2 R5** | 自審：public landing / quota fail-closed？ | 3 個 | 3 |
| **L2 R6** | promo 系統開發後自審 | union check / quota 保護 / rollback recovery | 3 |
| **L2 R7** | cross-system 自審 | 6 個 (1 High + 5 Important) | 6 |
| **L2 R8** | whole-folder re-check | 7 個 (1 Critical UX + 4 Important + 2 Minor) | 7 |
| **L2 R9** | 最後 deep sweep（R9B01-R9B08） | Recur retry double-grant / 退費邊界 / TWD / 時區 | ~30 |

**總修了 70+ 個 finding，PAYUNi 一個都沒指出來**。

→ 這代表「PAYUNi 過審」是地板，不是天花板。

## L2 自我加固的價值

跑 L2 自審找到的、PAYUNi 沒抓但會在 prod 出事的真實案例：

1. **R9B01 — Recur retry double-grant**：webhook 4xx 時 Recur 會 retry，dedup table 有 row 但 dispatch 沒做完 → 用戶永遠沒入點 → 客訴
2. **R9B07 — 退費 10-credit 邊界**：退費公式跨整數邊界算錯，退少 1 點 → 對帳會發現
3. **R9B06 — 加購包 vs 訂閱法律矛盾**：policy 寫訂閱可退，但加購包沒寫 — 用戶買加購包後要求退款會吵架
4. **R4B01 — Account deletion race**：刪除中 webhook 進來入點，secondary collections 留 orphan row → 半年後 DB 一堆鬼資料
5. **R4B05 — Auth/session race**：rate limiter 跟 session 沒同步 → 暴力破解門戶

PAYUNi 看不到這些（他們只測 demo flow），但 prod 出事一樣是你扛。

## Dual-AI Self-Audit 跑法

過 L1 後（送審通過 / sandbox 端到端跑得通），就可以開始跑 L2：

### 流程
1. **挑一個 module / area**：如 `account deletion 全鏈路` / `webhook handler` / `promo system`
2. **AI 1 跑審查**：餵相關檔案 + prompt：「以 PAYUNi 訂閱系統的角度，找 race condition / idempotency / barrier / dedup 漏洞。每個 finding 給：嚴重度、攻擊路徑、修法。」
3. **AI 2 跑同樣 prompt**（另一個 model）
4. **比對**：
   - 兩邊都認 → 一定要修（commit message `fix(audit-rN): 兩 AI 雙確認的 X 個 critical/important`）
   - 只有一邊認 → 看細節，多半 false positive
   - 兩邊都漏 → 把已知 finding 餵回去問「為什麼這個你沒抓到」找盲點
5. **修完 → 下一個 module**

### 「兩 AI 雙確認」commit message 慣例
```
fix(audit-r4b01): 帳號刪除 race conditions — 兩 AI 雙確認的 4 個 critical/high
fix(audit-r9b01): Flask tuple crash + Recur retry double-grant + tz race + verify CAS
fix(audit-r9b07): refund 10-credit 邊界對齊 + 系統故障退費公式 + TWD 聲明 + test env setdefault
```

格式：`fix(audit-rN[bM]): {精簡內容} — {finding 統計}`
- `rN` = round number
- `bM` = sub-batch（同 round 多個模組分批）
- 後綴統計：finding 數量 / 嚴重程度

→ 三個月後回頭看 git log 知道「這段時間在做 self-audit 加固」，不是亂改。

### 工具化版本
[dual-audit skill](../dual-audit/SKILL.md) 是這個流程的自動化版本。手動跑就照上面 1-5 步。

## L2 該掃的範圍

L1 通過後，照這個順序跑 L2：

| 優先順序 | 模組 | 為什麼這順序 |
|---|---|---|
| 1 | Account deletion 全鏈路 | 個資法 + DB orphan 風險高 |
| 2 | Webhook dispatch | retry / double-grant 一發生就客訴 |
| 3 | Credits / quota atomic | 套利風險 |
| 4 | Promo / 兌換碼 | 跟訂閱 union check 容易漏 |
| 5 | Public endpoints | 沒登入入口 fail-closed 要嚴 |
| 6 | Bridge / 外部 API 整合 | 跨 process race 多 |
| 7 | Scheduler / cron | ghost-send / 時區地雷 |
| 8 | Admin routes | 後台保護 |

每個模組過完 dual-AI 就有信心。

## L1 vs L2 該優先做哪個

**送審 deadline 緊**（PAYUNi 那邊催）：先做 L1 過審，L2 慢慢補
**送審還早**（自己時程鬆）：做 L1 + L2 R1-R3 一起送（過審證據更扎實，PAYUNi 看 commit log 也會留好印象）

無論順序，**L2 別跳過**。送審只是入場券，prod 穩定才是真的。

## 心理建設

- **L1 一次過審是常態**，照本 skill 走，refund ledger / deletion / Portal / policy 一致這 4 項做扎實就過
- **L2 找到 90+ finding 也是常態**，不代表你 L1 做得爛 — 代表 PAYUNi 看不到的角落很多
- **每輪 L2 R1-R9 都會引發下一輪**：修了 deletion barrier → 發現 secondary collections 也要 barrier → 發現 scheduler 也要 → 發現 bridge webhook 也要 ... 大概 5-8 輪會收斂
- **commit 用 `fix(audit-rN)` 格式**：之後再回審 / 出問題回查很有用

## 過審後（半年週期）

- PAYUNi 定期 review（大概半年一次）
- 大改版時要報備（PAYUNi 後台「變更通知」）
- 新功能要更新註冊營業項目
- 退款率 / chargeback 超閾值會被請求說明
- → 每次大改版就**重跑一次 L2 dual-AI self-audit**，commit log 留證據
