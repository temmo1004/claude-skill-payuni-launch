# PAYUNi 送審常見 Audit Findings + 修法

整理過審時 PAYUNi 真實提出的 finding + 對應 fix recipe。每個 finding 標 P0 / HIGH / MEDIUM，**P0 不修不過**。

完整 reference 實作見：
- Refund / Account deletion / Customer_id-first → `webhook-recipe.md` + `account-deletion.md`
- Customer Portal → `customer-portal.md`
- DB schema → `db-schema.md`

---

## P0 Blockers（不修不過）

### F1: Refund ledger 缺失
**Finding 原文**：「退款流程缺乏可追溯記錄，無法配合對帳」

**為什麼是 P0**：PAYUNi 對帳機制依賴每筆退款 DB 有 row；沒這 ledger 出包時無法佐證。

**修法**：
```python
# DB schema
col_refund_ledger.create_index("event_id", unique=True)

# webhook handler 加 refund 分支
def _handle_refund(event, data):
    col_refund_ledger.insert_one({
        "event_id": event["id"],
        "ts": datetime.now(timezone.utc),
        "amount": data.get("amount"),
        "currency": data.get("currency", "TWD"),
        "user_id": str(u["_id"]) if u else None,
        "customer_id": data.get("customer", {}).get("id"),
        "original_order_id": data.get("order_id"),
        "status": event_type,
        "raw": event,
    })
    # 還要扣回入點（防套利）
```

**完整流程**：`webhook-recipe.md` § Python `_handle_refund`

**驗收**：
- 跑 sandbox refund 一次，DB 看得到 `col_refund_ledger` 有 row
- `SELECT COUNT(*) FROM refund_ledger` ≥ Recur dashboard 顯示退款數

---

### F2: 帳號刪除功能缺失
**Finding 原文**：「使用者無法自行刪除帳號，不符合個資法第 11 條」

**為什麼是 P0**：個資法強制；審查員會在 `/billing` 找不到刪除按鈕直接退件。

**修法**：
- `/api/account/delete` endpoint（密碼確認 + rate limit）
- UI 入口在 `/billing` 頁底（**顯眼，紅色按鈕**）
- Soft-delete 流程：
  1. CAS 拿 deletion lock，設 `deletion_started_at`（barrier）
  2. **先取消 Recur 訂閱**（call cancel API），失敗回 502
  3. Pre-sweep secondary collections，任一失敗 → ABORT
  4. Anonymize `credits_ledger`
  5. Delete user doc → 「關門」
  6. Post-anonymize ledger
  7. Second-pass sweep race window 內 row
- Refund ledger **不刪只 anonymize**（法定 5 年保留）

**完整流程**：`account-deletion.md`

**驗收**：
- demo account 訂閱後在 `/billing` 點刪帳號 → 完整跑完
- DB 看 user doc 消失、ledger anonymize、refund_ledger 仍在
- Recur dashboard 看訂閱已 cancelled

---

### F3: Customer 對應靠 email（脆弱）
**Finding 原文**：「webhook 用 email 對應使用者，使用者改 email 後 entitlement 會失聯」

**為什麼是 P0**：用戶在 PAYUNi 端改 email 後就拿不到入點，客訴必爆。

**修法**：entitlement dispatch 用「customer_id-first」：
```python
def _resolve_user(event):
    customer = event["data"]["customer"]
    customer_id = customer.get("id")
    # 1) 優先用 customer_id
    if customer_id:
        u = col_users.find_one({"recur_customer_id": customer_id})
        if u: return u
    # 2) fallback email + 寫回 customer_id（self-heal）
    email = (customer.get("email") or "").lower()
    if email:
        u = col_users.find_one({"email": email})
        if u and customer_id:
            col_users.update_one(
                {"_id": u["_id"]},
                {"$set": {"recur_customer_id": customer_id}},
            )
        return u
    return None
```

**完整流程**：`webhook-recipe.md` § `_resolve_user`

**驗收**：
- 註冊 → 訂閱 → 在 PAYUNi 端改 email → 等下一個 webhook → DB user row 仍對應到正確 customer_id

---

## HIGH（4 個常見）

### F4: 取消承諾 vs UI 落空
**Finding 原文**：「Terms 寫可取消訂閱，但 UI 找不到取消按鈕」

**為什麼是 HIGH**：屬於「網站宣稱 vs 實際做得到」的矛盾，審查員必抓。

**修法**：接 Recur Customer Portal
```python
@app.route("/api/billing/portal", methods=["POST"])
@login_required
def api_billing_portal():
    user = _current_user()
    customer_id = user.get("recur_customer_id")
    if not customer_id:
        return jsonify({"error": "no_customer_id"}), 400
    r = requests.post(
        f"{RECUR_API_BASE}/v1/billing/portal-sessions",
        headers={"Authorization": f"Bearer {RECUR_SECRET_KEY}"},
        json={"customer_id": customer_id,
              "return_url": f"{APP_BASE_URL}/billing?from=portal"},
    )
    return jsonify({"ok": True, "url": r.json()["url"]})
```

UI：`<button id="manage-btn">管理訂閱 / 取消 / 發票</button>` 放在 `/billing` 顯眼位置

**完整流程**：`customer-portal.md`

**驗收**：點按鈕跳轉 Recur portal → 可以實際取消訂閱 → 跳回後 `/billing` UI 反映取消狀態

---

### F5: AI 使用聲明在 Terms / Privacy 不一致
**Finding 原文**：「Terms 寫『不使用 AI 自動處理』，但 Privacy § 5 寫『使用 LLM 生成內容』」

**為什麼是 HIGH**：兩份文件對 AI 範圍說不同 → 法律承諾不一致。

**修法**：兩份文件對齊。
- Privacy § 7 寫明 AI 使用範圍（哪個功能用 / 用哪家模型 / 是否拿去訓練）
- Terms § 6 直接引用 Privacy § 7，**不要自己另寫一遍**

✗ **不一致範例**：
- Privacy：「使用 OpenAI GPT-4 生成行銷文案」
- Terms：「本服務不使用 AI 自動處理」

✓ **一致範例**：
- Privacy § 7：「本服務在『生成行銷文案』功能使用 LLM；輸入不用於模型訓練」
- Terms § 6：「AI 使用範圍請參閱《隱私權政策》§ 7」

**驗收**：grep `AI`、`LLM`、`自動` 兩份文件，沒矛盾

---

### F6: LINE 相關宣稱誇大（適用任何串接第三方平台的服務）
**Finding 原文**：「網站寫『100% 即時送達』，LINE 平台本身不保證」

**為什麼是 HIGH**：宣稱無法兌現 → 消保法。

**修法**：軟化用詞
| ❌ 別寫 | ✓ 改寫 |
|---|---|
| 100% 即時送達 | 以 LINE 平台為投遞通道 |
| 保證即時 | 投遞依 LINE 平台狀態 |
| 永不漏發 | 失敗會自動 retry 3 次 |
| 24/7 不中斷 | 我們會盡力維持服務 |
| 完美整合 | 與 LINE 平台 API 整合 |

通用模板（任何第三方依賴）：
> 本服務透過 {第三方平台} 提供 {功能}。{第三方平台} 的服務品質非本服務可控；
> 若 {第三方平台} 變更或中斷服務，本服務將盡力配合調整但不另行賠償。

**驗收**：grep `100%`、`保證`、`絕對`、`永遠`、`完美` policy 頁，全清掉

---

### F7: 範圍模組混淆
**Finding 原文**：「網站定位為房仲工具，但有保險推銷功能 — 範圍不一致」

**為什麼是 HIGH**：PAYUNi 註冊的「營業項目」≠ 網站實際功能 → 規避審查嫌疑。

**修法**：砍掉非核心模組
```bash
# 直接砍模組目錄
git rm -r modules/insurance/
# 同時砍 routes / nav menu / settings 提到的入口
grep -r "insurance" src/ templates/  # 全清
```

**注意**：要保留 DB 中既有 module 資料的 deletion 路徑（不能變成 zombie row）。
舊用戶的 module data 在 account deletion 時要一併清。

**驗收**：navigation / footer / sitemap.xml 不再出現非核心模組

---

## MEDIUM

### F8: DB unique index 缺失
**Finding 原文**：「公開資源 URL 用 slug，但無 unique index — 可能撞名」

**修法**：
```python
# Mongo
col_resources.create_index("slug", unique=True, sparse=True)
col_processed_events.create_index("event_id", unique=True)
col_refund_ledger.create_index("event_id", unique=True)

# Postgres
ALTER TABLE resources ADD CONSTRAINT slug_unique UNIQUE(slug);
```

詳見 `db-schema.md`

**驗收**：嘗試 insert duplicate → 拋 DuplicateKeyError

---

### F9: support email 用 gmail
**Finding 原文**：「客服 email 為 gmail，不符合企業形象」

**修法**：
- 自有域名 + Cloudflare Email Routing → 轉發到 gmail（收信）
- ZSend / SendGrid 寄信用 `noreply@yourdomain.com`
- 改 `policy_contact.html` 顯示 `support@yourdomain.com`

詳見 `deployment-tips.md` § Email Infra

**驗收**：寄信到你的 `support@yourdomain.com` → 30 秒內到 gmail

---

### F10: 品牌一致性
**Finding 原文**：「不同頁顯示的產品名不統一」

**修法**：所有 display surface 加品牌後綴
```html
<title>{產品名} ＿{你的品牌}</title>
<h1>{產品名} ＿{你的品牌}</h1>
```

或統一前綴：
```html
<title>{你的品牌} · {頁面名}</title>
```

範例：
- ❌ 首頁 title「房仲工具」、登入頁 title「LineMarketing」、`/billing` 顯示「實價登錄系統」
- ✓ 全部 title 統一「房仲工具 ＿{品牌}」

**驗收**：grep 全部 `<title>` + `<h1>`，命名一致

---

## 你還可能遇到的（推測）

### F11: 隱私政策更新通知機制
**Finding（推測）**：「Privacy 更新如何通知用戶？」

**修法**：Privacy 頁底加：
```
本政策最後更新日期：YYYY-MM-DD
重大變更時將透過註冊 email 通知；持續使用本服務即視為同意更新內容。
```

### F12: 訂閱方案明列價格與週期
**Finding（推測）**：「訂閱頁未明確顯示扣款週期 / 自動續訂」

**修法**：`/billing` 每個方案顯示：
- 價格（NT$XXX / 月 or / 年）
- 「自動續訂，可隨時取消」
- 取消後 period_end 失效

### F13: 未成年人保護
**Finding（推測）**：「未限制未成年人付款」

**修法**：Terms § 2 加：「使用本服務須滿 18 歲」+ checkout 前彈窗確認

---

## 自審清單（送審前跑一遍）

每個 finding 對應一條：

1. **退款**：每筆退款是否在 DB 有 row？SQL/Mongo 查得出來？
2. **刪帳號**：點 `/billing` 找得到刪帳號按鈕嗎？刪了會清乾淨嗎？
3. **Webhook 對應**：使用者改 email 後 entitlement 還會到位嗎？（測 customer_id-first）
4. **取消訂閱**：UI 上點得到「取消訂閱」按鈕嗎？實際能取消嗎？
5. **AI 聲明**：Terms 跟 Privacy 對 AI 使用範圍一致嗎？grep 過了？
6. **平台宣稱**：有沒有寫「100% / 保證 / 永遠 / 絕對」這種話？
7. **範圍**：網站功能 = PAYUNi 註冊的營業項目嗎？多餘的有沒有砍乾淨？
8. **DB 完整性**：slug / event_id / unique 欄位有沒有 index？
9. **客服**：email 是自有域名嗎？實際能收信嗎？（寄一封測試）
10. **品牌**：所有頁面的產品名、tagline、footer 都一致嗎？
11. **發票**：policy 寫的發票方式跟你 PAYUNi 註冊身份一致嗎？（個人不能寫「可開發票」）
12. **語言**：中英文 routing 都有 mirror 嗎？英文 policy 跟中文等價嗎？

打完勾再送審。
