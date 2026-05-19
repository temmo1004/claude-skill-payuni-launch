# payuni-launch

## 簡介
台灣 SaaS 從 0 到 PAYUNi（統一金流）**一次過審** + 接 Recur 收款的 playbook。Stack-agnostic — 提供 Python / Node、Mongo / Postgres 變體。基於 2026-04 一次過審實戰反推，後續以多專案抽象化。

## 觸發條件
- 觸發詞：`payuni`、`金流送審`、`送審準備`、`Recur 接金流`、`統一金流`、`subscription billing 接台灣金流`
- 使用場景：
  - 新專案要送 PAYUNi 商家審核
  - 用 Recur（SaaS 訂閱層）接 PAYUNi
  - 需要快速搭出 policy 頁 / 退款 ledger / 帳號刪除 / Customer Portal
  - 既有專案要補合規漏洞

## 為什麼存在這個 skill
PAYUNi 審查像通關密語：知道要寫什麼就一次過，不知道就反覆來回 1-2 週。實戰過審後反推：審查員不是讀政策，是**比對網站宣稱 vs 實際 UI vs DB 行為**有沒有矛盾。這 skill 把所有「矛盾陷阱」拆給你看。

## 跨專案使用方式

### 基本流程
```
/payuni-launch              # 完整 6 天 sprint
/payuni-launch infra        # 只跑 Phase 1（架構鋪底）
/payuni-launch policy       # 只跑 Phase 2（政策頁）
/payuni-launch audit        # 只跑 Phase 3（audit finding 修復）
/payuni-launch checklist    # 列 18 項勾選清單
```

### 第一次跑這個 skill：先回答 6 個問題
你的專案的這些值會貫穿整個 sprint，先寫在便箋上（或讓 AI 幫你問）：

| 變數 | 範例 | 說明 |
|---|---|---|
| `{BRAND}` | `your-product` | 內部代號 |
| `{DISPLAY}` | `Your Product ＿YourBrand` | UI 顯示名（給審查員看的） |
| `{DOMAIN}` | `yourdomain.com` | 自有域名（**不能 gmail**） |
| `{STACK}` | `flask+mongo` / `fastapi+postgres` / `nextjs+prisma` | 後端 stack |
| `{MERCHANT_TYPE}` | `個人` / `公司` / `行號` | PAYUNi 註冊身份（決定能否開發票） |
| `{HAS_LINE}` | `true` / `false` | 是否串接 LINE（會影響 Privacy § 8） |
| `{HAS_AI}` | `true` / `false` | 是否用 LLM 處理使用者資料（影響 AI 聲明） |

之後文件 / code 範本裡所有 `{}` 都對應這些變數，找替換即可。

## 4 個 Phase 結構

### Phase 1 — 架構鋪底（Day 1-2）
詳見 `webhook-recipe.md` + `checkout-integration.md` + `db-schema.md`。

**必備元件：**
1. **Recur webhook endpoint** — `POST /api/webhooks/recur`
   - HMAC-SHA256 + Base64 簽章 + `compare_digest`
   - Dedup：`processed_events` 用 `event_id` unique index
   - Fail-closed：`RECUR_WEBHOOK_SECRET` 未設 → **503 不接事件**（不要 fallback skip）
   - Dispatch 失敗 → **500**（觸發 Recur 重試）
   - Audit log：原始 payload → `recur_events` collection
2. **Checkout** — Hosted（recommended）或 Modal
   - Hosted：redirect 走 `?status=success` → polling 等 webhook
   - Modal：要先收 email 才能 `subscribe()`
3. **訂閱 / 案點 entitlement 層**
   - Balance 顯示
   - Webhook → 入點 dispatch
   - Paywall guard（餘額不足擋 API）

### Phase 2 — Policy 頁（Day 3-4）
詳見 `policy-templates.md`。

**必備頁面（中英雙語各 1）：**
- `/policy/privacy` — 必含 § 4 cookie、§ 5 用戶權利、§ 7 AI 聲明、§ 8 LINE（如有）
- `/policy/terms` — 跟 Privacy AI / LINE 條款**一字不差**
- `/policy/refund` — **PAYUNi 強制檢查**
- `/policy/payment-info` — 寫明 PAYUNi 處理 + 不存卡
- `/policy/contact` — 自有域名 email + 服務時間
- 首頁 footer 加全部 policy 連結

### Phase 3 — Audit response（Day 5）
詳見 `audit-findings.md`。

**P0 blocker（不修不過）：**
1. Refund ledger — 每筆退款 DB 留可追溯 row
2. Account deletion — 自助刪除 + secondary cleanup + deletion barrier
3. Customer_id-first lookup — webhook 對應使用者先 customer_id 後 email

**HIGH：**
- Customer Portal — 修「Terms 寫可退訂但 UI 沒按鈕」
- AI 聲明對齊 — Terms ↔ Privacy 一致
- LINE 用詞軟化 — 不寫「100% / 保證 / 永不」
- 範圍收斂 — 砍掉跟主定位無關的模組

**MEDIUM：**
- DB unique index（slug、event_id、recur_customer_id）
- 自有 email 域名
- 品牌名跨頁一致

### Phase 4 — 收尾（Day 6）
- 主導覽列加 billing 入口（**讓審查員找得到付款頁**）
- `/billing` 補帳號刪除 UI
- 所有 display surface 加品牌後綴
- 中英文 routing 補完（`/en/*` 全對應）

## 補充檔案
| 檔案 | 內容 |
|---|---|
| `checklist.md` | 18 項勾選清單（按 Phase 排序） |
| `policy-templates.md` | 6 份 policy 頁中英骨架 |
| `audit-findings.md` | P0/HIGH/MEDIUM 10 個 finding + 修法 |
| `webhook-recipe.md` | Recur webhook 完整實作（Python / Node） |
| `account-deletion.md` | 帳號刪除 + deletion barrier pattern |
| `customer-portal.md` | Recur Customer Portal 整合 |
| `checkout-integration.md` | Hosted vs Modal Checkout + 付款後 polling |
| `db-schema.md` | 必備 collections / tables（Mongo + Postgres） |
| `taiwan-tax-law.md` | 統一發票義務 / ezPay 電子發票 |
| `deployment-tips.md` | Zeabur / Vercel / Railway 部署環境變數 |

## 法律基礎（台灣）
詳見 `taiwan-tax-law.md`。

**速查：**
- 個人月銷 ≤ NT$40K → 不開發票
- NT$40K-200K → 小規模 1% 季繳
- ≥ NT$200K → 一般營業人 5% VAT，每筆電子發票
- 個人 PAYUNi 帳號**沒統編、不能開 e-invoice** — policy 頁不能寫「可開發票」

## 依賴
- Recur SaaS account（`app.recur.tw`）
- PAYUNi 商家帳號（個人 / 公司 / 行號）
- 自有 email 域名（推薦 Cloudflare Email Routing + Zeabur ZSend / SendGrid）
- 任一資料庫支援 unique index（Mongo / Postgres / MySQL）

## 過審心法（一句話）
> 審查員不是讀政策，是抓**矛盾**：Terms 寫的 vs UI 做得到的 vs DB 真實寫入的 vs PAYUNi 註冊資料的 — **四者必須一致**。

跑這個 skill 的所有 phase，就是在消除這四者之間每一條可能的矛盾。

## 更新日誌
- 2026-05-19 v0.1 — 初版，從單一專案反推
- 2026-05-19 v0.2 — 跨專案抽象化、補 webhook / deletion / portal / schema reference
