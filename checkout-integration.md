# Checkout 整合（Hosted vs Modal）

Recur 提供兩種 checkout：

| 模式 | UX | 整合難度 | 適合 |
|---|---|---|---|
| **Hosted Checkout**（**推薦**）| 跳轉到 Recur 託管頁，付完 redirect 回來 | 簡單 | 大多情境，過審首選 |
| **Modal Checkout** | popup overlay 不離開原頁 | 較難（要先收 email） | 想保留品牌氛圍 |

## 為什麼推薦 Hosted
- 不用 SDK，純後端 + redirect URL 搞定
- 信用卡輸入發生在 Recur 域名 → 你不碰卡號，PCI 範圍縮到最小
- PAYUNi 審查直接認 Recur hosted page

---

## Hosted Checkout — 完整流程

### 後端

```python
# /api/billing/config — 給前端讀基本資訊
@app.route("/api/billing/config")
def billing_config():
    return jsonify({
        "publishable_key": os.environ["RECUR_PUBLISHABLE_KEY"],
        "products": [
            # whitelist 你想公開的 product
            {"id": "prod_basic", "name": "Basic", "price": 199, "credits": 100},
            {"id": "prod_pro", "name": "Pro", "price": 499, "credits": 300},
        ],
    })


# /api/billing/status — 給前端讀當前訂閱 + 點數
@app.route("/api/billing/status")
@login_required
def billing_status():
    user = _current_user()
    return jsonify({
        "ok": True,
        "credits": {
            "total": user.get("credits", 0),
        },
        "subscription": {
            "status": user.get("subscription_status"),
            "product_id": user.get("subscription_product_id"),
            "period_end": user.get("subscription_period_end"),
        },
    })


# 建立 checkout session（後端 call Recur）
@app.route("/api/billing/checkout", methods=["POST"])
@login_required
def billing_checkout():
    user = _current_user()
    product_id = (request.get_json() or {}).get("product_id")
    if product_id not in ALLOWED_PRODUCT_IDS:
        return jsonify({"error": "invalid_product"}), 400

    r = requests.post(
        f"{RECUR_API_BASE}/v1/checkout/sessions",
        headers={"Authorization": f"Bearer {RECUR_SECRET_KEY}"},
        json={
            "product_id": product_id,
            "customer_email": user["email"],
            "customer_id": user.get("recur_customer_id"),  # 有則帶上
            "success_url": f"{APP_BASE_URL}/billing?status=success",
            "cancel_url": f"{APP_BASE_URL}/billing?status=cancelled",
            "metadata": {
                "user_id": str(user["_id"]),  # 重要：webhook 時可用
            },
        },
        timeout=10,
    )
    if r.status_code >= 400:
        return jsonify({"error": "checkout_unavailable"}), 502
    return jsonify({"ok": True, "url": r.json()["url"]})
```

### 前端 `/billing` 頁

```html
<div id="products" class="grid gap-4"></div>

<div id="status-section" class="mt-6">
  <h3>當前狀態</h3>
  <p>點數：<span id="credits-total">0</span></p>
  <p id="credits-detail" class="text-sm text-gray-500"></p>
  <p>訂閱：<span id="sub-status">未訂閱</span></p>
</div>

<script>
async function fetchAndRenderStatus() {
  const r = await fetch('/api/billing/status', { credentials: 'same-origin' });
  if (!r.ok) return null;
  const d = await r.json();
  document.getElementById('credits-total').textContent = d.credits?.total ?? 0;
  document.getElementById('sub-status').textContent = d.subscription?.status || '未訂閱';
  return d;
}

async function renderProducts() {
  const r = await fetch('/api/billing/config');
  const d = await r.json();
  document.getElementById('products').innerHTML = d.products.map(p => `
    <div class="border rounded p-4">
      <h4 class="font-bold">${p.name}</h4>
      <p>NT$${p.price} / ${p.credits} 點</p>
      <button onclick="buy('${p.id}')" class="bg-brand-600 text-white px-4 py-2 rounded">
        購買
      </button>
    </div>
  `).join('');
}

async function buy(productId) {
  const r = await fetch('/api/billing/checkout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ product_id: productId }),
  });
  const d = await r.json();
  if (d.ok) {
    window.location.href = d.url;  // 跳到 Recur hosted page
  } else {
    alert(d.error || '無法開啟結帳，請稍後再試');
  }
}

// ★★★ 關鍵：付款成功後 polling 等 webhook ★★★
document.addEventListener('DOMContentLoaded', async () => {
  const fromSuccess = new URLSearchParams(location.search).get('status') === 'success';
  const data = await fetchAndRenderStatus();
  await renderProducts();

  if (fromSuccess) {
    const baseTotal = data?.credits?.total ?? 0;
    const baseSub = data?.subscription?.status ?? null;
    document.getElementById('credits-detail').textContent = '付款已完成，正在入點⋯';

    // 8 次 × 1.5s = 最多等 12 秒
    for (let i = 0; i < 8; i++) {
      await new Promise(r => setTimeout(r, 1500));
      const d2 = await fetchAndRenderStatus();
      const t2 = d2?.credits?.total ?? 0;
      const s2 = d2?.subscription?.status ?? null;
      if (t2 !== baseTotal || s2 !== baseSub) break;  // 變了，webhook 來了
    }
    document.getElementById('credits-detail').textContent = '';

    // 清掉 URL query，避免 refresh 重觸發
    try { history.replaceState({}, '', '/billing'); } catch (e) {}
  }
});
</script>
```

### 為什麼要 polling

| 時序 | 事件 |
|---|---|
| T+0.0s | 用戶在 Recur hosted page 按「確認付款」 |
| T+0.5s | Recur 處理付款成功，redirect 用戶到 `?status=success` |
| T+0.6s | 用戶瀏覽器抵達 `/billing`，fetch `/api/billing/status` |
| T+1.0s | Recur 觸發 webhook 通知你的後端 |
| T+1.2s | 你的 webhook handler 入點到 DB |

→ 用戶在 **T+0.6s** 看到的點數還是舊的。**沒 polling 就只能叫他按 F5**。

8 次 1.5 秒間隔的 polling 在 12 秒內捕捉 webhook 入點，足夠 cover 95%+ 的延遲。

---

## Modal Checkout — 變體（須先收 email）

⚠️ **重要**：Modal 要求 `customerEmail` 在 `subscribe()` 前先傳，否則 SDK 會直接 fail。

### React SDK

```tsx
import { RecurProvider, useSubscribe } from 'recur-tw';

function App() {
  return (
    <RecurProvider publishableKey={process.env.NEXT_PUBLIC_RECUR_KEY}>
      <CheckoutFlow />
    </RecurProvider>
  );
}

function CheckoutFlow() {
  const { subscribe, isLoading, error } = useSubscribe();
  const [email, setEmail] = useState('');

  async function handleBuy(productId: string) {
    if (!email) {
      alert('請先輸入 email');
      return;
    }
    await subscribe({
      product_id: productId,
      customer_email: email,  // ← Modal 必填
      success_url: `${location.origin}/billing?status=success`,
    });
  }

  return (
    <>
      <input
        type="email"
        value={email}
        onChange={e => setEmail(e.target.value)}
        placeholder="email"
        required
      />
      <button onClick={() => handleBuy('prod_basic')} disabled={isLoading}>
        購買 Basic
      </button>
      {error && <p className="text-red-600">{error.message}</p>}
    </>
  );
}
```

---

## Webhook → 入點的 dispatch（後端）

```python
# 在你的 webhook handler 加：
def _handle_payment(event, data):
    u = _resolve_user(event)
    if not u or u.get("deletion_started_at"):
        return

    # metadata 裡的 user_id 是最可靠來源（你在 checkout 時帶上）
    metadata = data.get("metadata", {})
    explicit_uid = metadata.get("user_id")
    if explicit_uid and str(u["_id"]) != explicit_uid:
        logging.warning("[webhook] uid mismatch %s vs %s", explicit_uid, u["_id"])

    product_id = data.get("product_id") or data.get("line_items", [{}])[0].get("product_id")
    credits = PRODUCT_CREDIT_MAP.get(product_id, 0)
    if credits > 0:
        col_users.update_one(
            {"_id": u["_id"], "deletion_started_at": {"$exists": False}},
            {"$inc": {"credits": credits}},
        )
        col_credits_ledger.insert_one({
            "user_id": str(u["_id"]),
            "event_id": event["id"],  # idempotency anchor
            "delta": credits,
            "reason": "purchase",
            "ts": datetime.now(timezone.utc),
        })


PRODUCT_CREDIT_MAP = {
    "prod_basic": 100,
    "prod_pro": 300,
}
```

---

## Paywall Guard — 點數不足擋 API

```python
def require_credits(amount=1):
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            uid = _current_user_id()
            user = col_users.find_one({"_id": ObjectId(uid)})
            if (user.get("credits", 0)) < amount:
                return jsonify({
                    "ok": False,
                    "error": "insufficient_credits",
                    "current": user.get("credits", 0),
                    "needed": amount,
                    "redirect": "/billing",
                }), 402
            # 原子扣點（防併發超發）
            result = col_users.update_one(
                {"_id": ObjectId(uid), "credits": {"$gte": amount}},
                {"$inc": {"credits": -amount}},
            )
            if result.matched_count == 0:
                return jsonify({"error": "credits_race"}), 402
            col_credits_ledger.insert_one({
                "user_id": uid,
                "delta": -amount,
                "reason": f"api:{request.path}",
                "ts": datetime.now(timezone.utc),
            })
            return f(*args, **kwargs)
        return wrapper
    return decorator

# 用法
@app.route("/api/generate-content", methods=["POST"])
@login_required
@require_credits(amount=1)
def generate_content():
    # ...
```

### 前端攔 402
```javascript
async function callApi(path, body) {
  const r = await fetch(path, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });
  if (r.status === 402) {
    const d = await r.json();
    if (confirm(`點數不足（剩 ${d.current}，需要 ${d.needed}）。前往儲值？`)) {
      window.location.href = d.redirect || '/billing';
    }
    return null;
  }
  return r.json();
}
```

---

## 測試流程

### Sandbox 端到端
1. 切 test mode (`pk_test_` / `sk_test_`)
2. `/billing` 點購買 → 跳 Recur hosted
3. 用測試卡 `4242 4242 4242 4242` + 任意 CVV / expiry
4. 完成 → redirect 回 `?status=success`
5. 確認 polling 跑了 → credits 變動 → URL 清乾淨
6. DB 看 `recur_events` 有事件、`credits_ledger` 有入點 row、`processed_events` 有 dedup key

### Prod cutover（test → live）
- [ ] 換成 `pk_live_` / `sk_live_`
- [ ] 換 webhook secret（live 跟 test 不同）
- [ ] 換 `RECUR_API_BASE` 如果 sandbox 用 staging URL
- [ ] 重啟服務（env var 不會 hot reload）
- [ ] 自己刷一張小額（NT$1 或最便宜方案）端到端跑一次
- [ ] 檢查 dashboard 有看到 live event

---

## 常見坑

| 坑 | 症狀 | 解 |
|---|---|---|
| 沒 polling，付完還是舊點數 | 用戶以為扣款失敗 | success polling pattern |
| 沒清 URL query | 用戶 F5 重新觸發 polling | `history.replaceState` |
| Product ID hardcode 在前端 | 改價要動 client code | `/api/billing/config` 給 whitelist |
| 沒 paywall guard | 點數扣到負數 | atomic `$gte: amount` filter |
| Test key 部署到 prod | 真實付款失敗 | env-name-prefixed key check 在 startup |
