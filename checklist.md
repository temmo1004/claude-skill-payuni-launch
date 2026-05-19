# PAYUNi 送審 18 項 Checklist

按 Phase 排序，每完成一項打勾，全部勾完才送審。每項標出**對應的 reference 檔**讓你深挖實作細節。

---

## Phase 1 — 架構鋪底（Day 1-2）

- [ ] **1. Recur webhook endpoint**
  - 路由：`POST /api/webhooks/recur`
  - HMAC-SHA256 + Base64 簽章 + constant-time compare
  - Dedup：`processed_events.event_id` unique index
  - Fail-closed：`RECUR_WEBHOOK_SECRET` 未設 → 503
  - Dispatch 失敗 → 500（讓 Recur 重試），並 rollback dedup row
  - Audit log：原始 payload → `recur_events` collection
  - 📖 完整實作：`webhook-recipe.md`

- [ ] **2. Hosted Checkout 頁**
  - `/billing` 中文 + `/en/billing` 英文
  - `GET /api/billing/config` — 回 publishable_key + product whitelist
  - `GET /api/billing/status` — 當前訂閱 + 點數
  - `POST /api/billing/checkout` — 建 Recur checkout session
  - 付款成功 polling（避免要 F5）
  - 📖 完整實作：`checkout-integration.md`

- [ ] **3. 案點 / 訂閱 entitlement**
  - DB schema：`users.credits`, `credits_ledger`
  - Webhook dispatch → 入點（atomic with deletion barrier）
  - Paywall guard（餘額不足擋 API 回 402）
  - 退款扣回點數（防套利）
  - 📖 完整實作：`checkout-integration.md` § Paywall Guard + `webhook-recipe.md` § `_handle_refund`

---

## Phase 2 — Policy 頁（Day 3-4）

- [ ] **4. Privacy Policy（隱私權政策）**
  - 路徑：`/policy/privacy` + `/en/policy/privacy`
  - 必含：資料蒐集範圍、第三方分享、cookie、用戶權利
  - § 7 AI 使用聲明（**跟 Terms 對齊**）
  - § 8 LINE 特別保護（如有接 LINE）
  - 📖 模板：`policy-templates.md` § 1

- [ ] **5. Terms of Service（服務條款）**
  - 路徑：`/policy/terms` + `/en/policy/terms`
  - AI 條款引用 Privacy § 7，**一字不差**
  - LINE / 第三方平台承諾要軟化（不寫「100% / 保證」）
  - 📖 模板：`policy-templates.md` § 2

- [ ] **6. Refund Policy（退款政策）**
  - **PAYUNi 強制檢查項**
  - 寫清楚：可退情境、不可退情境、處理時程、聯絡方式
  - 訂閱要寫「period end 後失效」(對應 Customer Portal 取消邏輯)
  - 📖 模板：`policy-templates.md` § 3

- [ ] **7. Payment Info（付款說明）**
  - **PAYUNi 強制檢查項**
  - 寫「由 PAYUNi 統一金流安全處理」「不儲存信用卡」
  - 列受支付方式（信用卡 / 超商等）
  - 發票方式跟你的 PAYUNi 註冊身份一致（個人寫「不開立」）
  - 📖 模板：`policy-templates.md` § 4 + `taiwan-tax-law.md`

- [ ] **8. Contact（客服資訊）**
  - email（**自有域名**，不是 gmail）
  - 服務時間
  - 📖 模板：`policy-templates.md` § 5

- [ ] **9. Index footer 政策連結**
  - 首頁 footer 跳得到全部 5 個 policy 頁
  - 中英文版各一份
  - 📖 模板：`policy-templates.md` § 6

---

## Phase 3 — Audit response（Day 5）

- [ ] **10. Refund ledger**
  - `refund_ledger` collection / table
  - `event_id` unique index
  - 每筆 refund webhook → 一筆 row（含 event_id, amount, user_id, customer_id, ts）
  - **不設 TTL** — 法定 5 年保留
  - 📖 完整實作：`webhook-recipe.md` § `_handle_refund` + `db-schema.md`

- [ ] **11. Account deletion**
  - `POST /api/account/delete`（密碼確認 + rate limit）
  - UI 入口在 `/billing` 頁底（顯眼，紅色按鈕）
  - CAS lock + deletion barrier
  - **先取消 Recur 訂閱才能刪**
  - Double-sweep secondary collections
  - 📖 完整實作：`account-deletion.md`

- [ ] **12. Customer_id-first lookup**
  - Webhook 對應使用者：先 `recur_customer_id` 後 email
  - email fallback 命中時自動寫回 customer_id（self-heal）
  - 📖 完整實作：`webhook-recipe.md` § `_resolve_user`

- [ ] **13. Customer Portal**
  - `POST /api/billing/portal` → Recur portal session URL
  - UI 按鈕「管理訂閱 / 取消 / 發票」放在 `/billing` 顯眼位置
  - return_url 帶 `?from=portal` 觸發前端 polling
  - 修「網站寫可退訂但實際做不到」audit risk
  - 📖 完整實作：`customer-portal.md`

- [ ] **14. AI / 第三方平台聲明一致性**
  - Terms § 6 跟 Privacy § 7 對 AI use 必須一致
  - 平台相關宣稱不能誇大（grep `100%` / `保證` / `絕對` / `永遠` → 全刪）
  - 📖 完整實作：`audit-findings.md` § F5 + F6

---

## Phase 4 — 收尾（Day 6）

- [ ] **15. 主導覽列入口**
  - 加「案點 / 訂閱」menu item
  - 讓 PAYUNi 審查員一眼能找到 billing 頁
  - 📖 對應 finding：`audit-findings.md` § F10 衍生

- [ ] **16. 砍範圍外模組**
  - 砍掉跟核心定位不符的功能（**範圍 = PAYUNi 註冊的營業項目**）
  - 同步砍 routes / nav menu / settings / sitemap 的入口
  - 保留 DB deletion 路徑（不留 zombie）
  - 📖 完整實作：`audit-findings.md` § F7

- [ ] **17. 品牌 + 客服 一致性**
  - 所有 display surface 加品牌後綴（`<title>` / `<h1>` / footer）
  - support email 換成自有域名（Cloudflare Email Routing + ZSend）
  - PAYUNi logo / 「Powered by PAYUNi」標示（如 PAYUNi 要求）
  - 📖 完整實作：`audit-findings.md` § F9 + F10 + `deployment-tips.md` § Email Infra

- [ ] **18. 中英文 routing 補完**
  - `/en/*` 路由全部對應
  - billing / policy / home 都要有英文版
  - 英文 policy 跟中文等價（同樣承諾、同樣條款）

---

## 送審前最後檢查（不勾不送）

- [ ] **跑一次 sandbox test 購買**，確認 webhook 全程觸發
- [ ] **用 Recur MCP `get_production_readiness_checklist`** → score ≥ 95%
- [ ] **點開所有 policy 連結**，確認都能開（curl 全部 200）
- [ ] **中英文切換無 broken link**
- [ ] **PAYUNi 後台填的網站 URL = 你 deploy 的 URL**
- [ ] **客服 email 能收信**（寄一封測試到 `support@yourdomain.com`，30 秒內到信箱）
- [ ] **DB 驗證 SQL 跑過**（refund_ledger 有 row、dedup 無重複、deletion_started_at 沒卡死的）
  - 詳見 `db-schema.md` § DB 驗證 SQL
- [ ] **環境變數 sanity check**：prod 沒有 `pk_test_` / `sk_test_`
  - 詳見 `deployment-tips.md` § Test → Live Cutover Checklist
- [ ] **跑端到端 demo**：註冊 → 訂閱 → 取消（透過 Portal）→ 刪帳號 — **整個流程能跑完**
- [ ] **發票文字跟 PAYUNi 註冊身份一致**（個人不寫「可開發票」）
  - 詳見 `taiwan-tax-law.md`

## 通用 12 條自審問題

跑這 12 題，每題回答得出來才送：

1. 每筆退款是否 DB 有 row？SQL 查得出來？
2. `/billing` 找得到「刪除帳號」按鈕嗎？真能刪嗎？
3. 用戶改 email 後 entitlement 還會到位嗎？
4. UI 上點得到「取消訂閱」按鈕嗎？點下去真能取消嗎？
5. Terms 跟 Privacy 對 AI 使用範圍一致嗎？
6. 有沒有寫「100% / 保證 / 永遠 / 絕對」這種詞？
7. 網站功能 = PAYUNi 註冊的營業項目嗎？多餘有沒有砍？
8. slug / event_id / unique 欄位有沒有 index？
9. 客服 email 是自有域名嗎？實際能收信嗎？
10. 所有頁面的產品名、tagline、footer 一致嗎？
11. 發票方式跟 PAYUNi 註冊身份一致嗎？
12. 英文 policy 跟中文等價嗎？

全部 yes → 送審。
