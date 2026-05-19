# 台灣稅務 / 統一發票義務

PAYUNi 商家身份決定能不能開統一發票。送審 policy 頁文字必須跟你實際身份一致 — **個人 PAYUNi 不能開發票，policy 寫「可開發票」直接矛盾**。

## 商家身份矩陣

| 身份 | PAYUNi 註冊 | 統編 | 可開電子發票 | 月銷上限 | 稅務義務 |
|---|---|---|---|---|---|
| **個人** | 身分證 + 個人銀行 | ❌ | ❌ | 無正式上限，但月銷 NT$40K 起國稅局可能查定 | NT$40K-200K 小規模 1% 季繳；NT$200K+ 須改設行號或公司 |
| **行號**（獨資 / 合夥） | 商業登記 + 行號銀行 | ✅ | ✅（ezPay 自動開） | 無 | 1% 小規模 或 5% 一般營業人 |
| **公司**（有限 / 股份） | 公司登記 + 公司銀行 | ✅ | ✅（ezPay 自動開） | 無 | 5% 一般營業人（每筆都要開電子發票） |

## 統一發票義務門檻（服務業）

依《統一發票使用辦法》+《加值型及非加值型營業稅法》：

| 月平均銷售額 | 身份要求 | 稅率 | 開立發票 |
|---|---|---|---|
| ≤ NT$40,000 | 個人 OK | 不課稅 | 不開立 |
| NT$40K - 80K | 個人可繼續 | 不課稅但需查定 | 不開立 |
| NT$80K - 200K | 須登記行號 | 1% 季繳（小規模營業人） | 不開立電子發票，由國稅局查定 |
| > NT$200K | 須一般營業人 | 5% VAT，按月或雙月申報 | **每筆都要開電子發票** |

**注意**：年銷售額會被當成核定依據 —
- 年銷 < NT$480K → 個人 OK（≈ 月 40K）
- 年銷 NT$480K - 2.4M → 應登記為小規模 1%
- 年銷 > NT$2.4M → 應登記為一般營業人 5%

## 開電子發票（ezPay）

ezPay 是台灣 e-invoice 服務（跟 PAYUNi 同集團，整合方便）。

**必要前置：**
1. 有統編（行號或公司）
2. 開通 ezPay 電子發票
3. 跟 PAYUNi / Recur 串接 → 扣款成功自動觸發 ezPay 開立

**整合方式：**
- Recur 後台連 ezPay → 設定 webhook
- 用戶在 `/billing` 填統編 + 抬頭（如 B2B 客戶）→ 存 user table
- 扣款成功 → Recur 通知 ezPay 開立 → 電子發票寄到用戶 email

```python
# 用戶填發票資訊 endpoint
@app.route("/api/billing/invoice-info", methods=["POST"])
@login_required
def save_invoice_info():
    data = request.get_json() or {}
    tax_id = (data.get("tax_id") or "").strip()
    company_name = (data.get("company_name") or "").strip()
    # 驗 8 碼統編
    if tax_id and not re.match(r"^\d{8}$", tax_id):
        return jsonify({"error": "invalid_tax_id"}), 400
    col_users.update_one(
        {"_id": ObjectId(_current_user_id())},
        {"$set": {
            "invoice_tax_id": tax_id or None,
            "invoice_company_name": company_name or None,
        }},
    )
    return jsonify({"ok": True})
```

```python
# 扣款 webhook 帶發票資訊給 Recur（Recur 再轉 ezPay）
def _handle_payment(event, data):
    u = _resolve_user(event)
    if not u: return
    # ... 入點邏輯 ...
    # 觸發 ezPay 開立（可能你的 Recur dashboard 已自動處理）
    if u.get("invoice_tax_id"):
        # B2B 三聯式發票
        invoice_payload = {
            "buyer_tax_id": u["invoice_tax_id"],
            "buyer_name": u["invoice_company_name"],
            "items": [...],
        }
    else:
        # B2C 二聯式發票 + 載具（手機條碼 / 自然人憑證 / 捐贈）
        invoice_payload = {
            "buyer_name": u.get("name", "個人"),
            "carrier_type": u.get("invoice_carrier_type"),
            "carrier_id": u.get("invoice_carrier_id"),
            "items": [...],
        }
```

## Policy 頁文字（依身份分支）

### 個人 PAYUNi 註冊者
**Refund Policy / Payment Info 寫法：**
```
本服務目前為個人經營，依《統一發票使用辦法》第 4 條，
月銷售額未達起徵點之服務業，不開立統一發票。
如需收據，請於 /billing 下載 PAYUNi 交易明細。
```

**❌ 不要寫的句子：**
- ✗「可開立統一發票」
- ✗「可開立 B2B 三聯式發票」
- ✗「請填寫統編」

### 公司 / 行號 PAYUNi 註冊者
**Refund Policy / Payment Info 寫法：**
```
本服務透過 ezPay 電子發票自動開立。
請於 /billing 填寫統編 / 抬頭（如需 B2B 發票）。
電子發票將於扣款成功後 24 小時內寄至註冊 email。
個人用戶可選擇手機條碼 / 自然人憑證 / 捐贈載具。
```

## 升級路徑（個人 → 行號 / 公司）

通常順序：
1. **MVP 階段（個人）**：用個人 PAYUNi，月銷 ≤ NT$40K，不開發票，policy 寫清楚
2. **驗證 PMF（NT$40K-200K）**：去當地國稅局登記**行號（獨資）**，拿統編，1% 小規模
3. **規模化（> NT$200K）**：再升級為**有限公司**，5% VAT，每筆電子發票

**換 PAYUNi 帳號**：個人換行號要重新註冊 PAYUNi、重審。會卡 1-2 週服務。
建議**直接登記行號 / 公司開始**，省事。

## 國稅局實務小知識

- **「服務業」起徵點是 NT$40K**（《加值型及非加值型營業稅法》第 23 條 + 《小規模營業人銷售額查定辦法》）
- **「買賣業」起徵點是 NT$80K** — SaaS 算服務業
- **查定**：你不申報，國稅局每季依你的營收估算稅額（1% 季繳）
- **逾起徵點不登記**：罰 1-10 倍漏稅額（《稅捐稽徵法》第 41 條）
- **PAYUNi 自動回報國稅局嗎？** — 部分情境會（B2C 大額）。最保險：自己每月對帳。

## 對網站文字的影響清單

| 頁面 | 個人身份要寫 | 公司 / 行號身份要寫 |
|---|---|---|
| `/policy/payment-info` | 「不開立統一發票」 | 「自動開立電子發票」 |
| `/policy/refund` | 「退款依交易明細處理」 | 「同步沖銷電子發票」 |
| `/billing` | 不顯示「統編 / 抬頭」欄位 | 顯示「統編 / 抬頭」欄位 |
| `/policy/contact` 公司資訊 | 「經營者：{真實姓名}」 | 「公司：{名稱}、統編：{}」 |
| Footer | 不寫統編 | 寫 `© 統編 12345678` |

## 法規參考
- 《統一發票使用辦法》第 4 條（不開立統一發票之事業）
- 《加值型及非加值型營業稅法》第 23 條（小規模營業人）
- 《營業人銷售額查定辦法》（國稅局查定 1% 稅）
- 《電子發票實施作業要點》（ezPay 適用）
- 《個人資料保護法》第 11 條（用戶刪除權）

## 與審查員的常見問答

> Q：審查員問「你怎麼確認年銷不超門檻？」
> A：「目前 MVP 階段，依《統一發票使用辦法》第 4 條由國稅局查定。若超過 NT$80K 月銷，會升級為小規模營業人並登記。」

> Q：審查員問「policy 寫不開發票，那 PAYUNi 對帳怎辦？」
> A：「PAYUNi 後台交易明細為對帳依據，refund_ledger 在我們 DB 留 5 年。」

> Q：審查員問「沒發票對企業用戶不友善？」
> A：「現階段目標 C 端用戶。B2B 上線時會升級為公司並接 ezPay。」
