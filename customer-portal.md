# Recur Customer Portal 整合

PAYUNi HIGH finding：「Terms 寫可取消訂閱，但 UI 找不到取消按鈕」。Recur 提供 Customer Portal — 一鍵 redirect 到 Recur 託管的訂閱管理介面，使用者可自助：
- 取消訂閱
- 更新付款方式
- 下載發票 / 收據
- 查看訂閱歷史

接 Portal 是最簡單的「修退訂落空」audit risk 的方式。

## Portal Session API 流程

```
[User] click "管理訂閱" button
         ↓
POST /api/billing/portal     ← 你的 endpoint
         ↓
你的 server 拿 user.recur_customer_id
         ↓
POST https://api.recur.tw/v1/billing/portal-sessions  ← Recur
         ↓
拿回 portal URL（限時、單次使用）
         ↓
回傳給前端 → window.location = url
         ↓
使用者操作完 → 跳回你設定的 return_url
```

---

## Python 實作

```python
import os
import requests
from flask import jsonify

RECUR_API_BASE = "https://api.recur.tw"
RECUR_SECRET_KEY = os.environ["RECUR_SECRET_KEY"]
APP_BASE_URL = os.environ["APP_BASE_URL"]  # e.g. https://yourdomain.com

@app.route("/api/billing/portal", methods=["POST"])
@login_required
def api_billing_portal():
    user = _current_user()
    customer_id = user.get("recur_customer_id")

    if not customer_id:
        # 用戶從沒結帳過 → 沒 customer_id → 沒 portal 可進
        return jsonify({
            "ok": False,
            "error": "no_customer_id",
            "hint": "請先完成至少一次付款後再管理訂閱",
        }), 400

    try:
        r = requests.post(
            f"{RECUR_API_BASE}/v1/billing/portal-sessions",
            headers={
                "Authorization": f"Bearer {RECUR_SECRET_KEY}",
                "Content-Type": "application/json",
            },
            json={
                "customer_id": customer_id,
                "return_url": f"{APP_BASE_URL}/billing",
            },
            timeout=10,
        )
    except Exception as e:
        logging.warning("[portal] upstream error: %s", e)
        return jsonify({"ok": False, "error": "upstream_error"}), 502

    if r.status_code >= 400:
        logging.warning("[portal] upstream %s: %s", r.status_code, r.text[:200])
        return jsonify({"ok": False, "error": "portal_unavailable"}), 502

    data = r.json()
    return jsonify({"ok": True, "url": data["url"]})
```

## Node 變體

```javascript
router.post('/api/billing/portal', requireAuth, async (req, res) => {
  const user = req.user;
  if (!user.recur_customer_id) {
    return res.status(400).json({
      ok: false,
      error: 'no_customer_id',
      hint: '請先完成至少一次付款後再管理訂閱',
    });
  }

  try {
    const r = await fetch(`${RECUR_API_BASE}/v1/billing/portal-sessions`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.RECUR_SECRET_KEY}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        customer_id: user.recur_customer_id,
        return_url: `${process.env.APP_BASE_URL}/billing`,
      }),
    });
    if (!r.ok) {
      return res.status(502).json({ ok: false, error: 'portal_unavailable' });
    }
    const data = await r.json();
    res.json({ ok: true, url: data.url });
  } catch (e) {
    res.status(502).json({ ok: false, error: 'upstream_error' });
  }
});
```

## Recur SDK 變體（更乾淨）

```javascript
// Node — recur-tw/server
import { Recur } from 'recur-tw/server';
const recur = new Recur({ secretKey: process.env.RECUR_SECRET_KEY });

router.post('/api/billing/portal', requireAuth, async (req, res) => {
  if (!req.user.recur_customer_id) {
    return res.status(400).json({ error: 'no_customer_id' });
  }
  const session = await recur.billingPortal.create({
    customer_id: req.user.recur_customer_id,
    return_url: `${process.env.APP_BASE_URL}/billing`,
  });
  res.json({ ok: true, url: session.url });
});
```

---

## UI — `/billing` 加管理按鈕

```html
<!-- 已有訂閱才顯示 -->
<section id="manage-section" class="hidden mt-6">
  <h3 class="font-bold mb-2">管理訂閱</h3>
  <p class="text-sm text-gray-600 mb-3">
    在 Recur 託管的安全頁面取消訂閱、更新付款方式、下載發票。
  </p>
  <button id="manage-btn" class="bg-brand-600 text-white px-4 py-2 rounded">
    管理訂閱 / 取消 / 發票
  </button>
</section>

<script>
async function loadStatus() {
  const r = await fetch('/api/billing/status', { credentials: 'same-origin' });
  const d = await r.json();
  if (d.subscription?.status && d.subscription.status !== 'canceled') {
    document.getElementById('manage-section').classList.remove('hidden');
  }
}

document.getElementById('manage-btn').addEventListener('click', async () => {
  const btn = document.getElementById('manage-btn');
  btn.disabled = true;
  btn.textContent = '載入中...';
  try {
    const r = await fetch('/api/billing/portal', {
      method: 'POST',
      credentials: 'same-origin',
    });
    const d = await r.json();
    if (d.ok && d.url) {
      window.location.href = d.url;
    } else {
      alert(d.hint || d.error || '無法開啟管理頁，請稍後再試');
      btn.disabled = false;
      btn.textContent = '管理訂閱 / 取消 / 發票';
    }
  } catch (e) {
    alert('連線錯誤，請稍後再試');
    btn.disabled = false;
    btn.textContent = '管理訂閱 / 取消 / 發票';
  }
});

loadStatus();
</script>
```

---

## 處理「取消後」回流

User 在 Portal 取消訂閱後會跳回 `return_url`。但取消事件是 **webhook 通知**（`subscription.deleted` / `subscription.updated`，幾秒延遲）。

→ 跟 checkout success polling 同樣的 race condition，解法見 `checkout-integration.md` 的 polling pattern：

```javascript
// /billing 頁載入時
const params = new URLSearchParams(location.search);
if (params.get('from') === 'portal') {
  // 短暫 polling 等 webhook 更新本地狀態
  await pollUntilStatusChange();
  history.replaceState({}, '', '/billing');
}
```

設 `return_url` 帶上 `?from=portal` 來觸發這段邏輯：

```python
"return_url": f"{APP_BASE_URL}/billing?from=portal",
```

---

## 測試

### Sandbox
1. 切到 test mode (`sk_test_`)
2. 用測試卡 `4242 4242 4242 4242` 完成一次訂閱
3. 點「管理訂閱」→ 確認跳到 Recur portal
4. 在 portal 取消訂閱
5. 跳回後等幾秒，confirm `subscription_status` 變 `canceled`

### Prod 過審 demo
PAYUNi 審查員可能會：
1. 註冊 demo account
2. 訂閱（用 PAYUNi 測試模式）
3. 點「管理訂閱」**← 這個按鈕一定要在 `/billing` 頁顯眼**
4. 在 portal 取消
5. 確認跳回後 UI 顯示「已取消」

按鈕**藏太深**直接被質疑「實際做不到」。建議放在 `/billing` 主要付款卡片下方。

---

## 常見坑

| 坑 | 症狀 | 解 |
|---|---|---|
| 沒 customer_id 卻給 portal 按鈕 | 點下去 400 | UI 先 check status，沒訂閱不顯示 |
| `return_url` 寫死 localhost | prod 點完跳到不存在的網址 | 從 env var 讀 `APP_BASE_URL` |
| 取消後 UI 沒更新 | 用戶以為沒成功 | polling pattern 等 webhook |
| portal session url 拿到後 cache | 過期 / 重用會 401 | 每次都重新 create session |
| Manage button 跟 Checkout button 樣式一樣 | 用戶誤點 | 分離視覺：訂閱用主色、管理用 outline |
