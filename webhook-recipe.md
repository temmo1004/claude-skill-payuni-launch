# Recur Webhook 完整實作

PAYUNi 透過 Recur webhook 通知付款 / 訂閱 / 退款事件。這個檔給你 2 個 stack 的 reference：Python (Flask/FastAPI) + Node (Express)。

## 核心原則（不分 stack）

1. **HMAC 驗章**：用 `HMAC-SHA256 + Base64`，**constant-time compare**（防 timing attack）
2. **Dedup**：`event_id` unique index，重複事件直接 200 OK 不重跑
3. **Fail-closed**：`WEBHOOK_SECRET` 未設或驗章失敗 → **不接事件**（503 / 401）
4. **Dispatch 失敗 → 500**：讓 Recur 重試（webhook 至少送一次，靠你 dedup）
5. **Audit log raw payload**：先寫進 DB 再 dispatch，方便事後重播 / 對帳

## DB schema 需要

```
processed_events    # dedup table，event_id unique
  - event_id (str, unique index)
  - first_seen_at (datetime)

recur_events        # audit log，原始 payload 全存
  - event_id (str, indexed)
  - ts (datetime)
  - type (str, indexed)
  - payload (json/dict)

refund_ledger       # 退款專用（P0 blocker）
  - event_id (str, unique index)
  - ts (datetime)
  - amount (int)
  - currency (str)
  - user_id (str, indexed)
  - customer_id (str, indexed)
  - original_order_id (str)
  - status (str)
  - raw (json/dict)
```

Postgres 變體見 `db-schema.md`。

---

## Python (Flask) — 完整實作

```python
import hmac, hashlib, base64, os, json, logging
from datetime import datetime, timezone
from flask import Blueprint, request, jsonify

bp = Blueprint("recur_webhook", __name__)

WEBHOOK_SECRET = os.environ.get("RECUR_WEBHOOK_SECRET")
SIGNATURE_HEADER = "X-Recur-Signature"  # 確認你 Recur dashboard 設的 header

def _verify_signature(raw_body: bytes, signature_b64: str) -> bool:
    if not WEBHOOK_SECRET or not signature_b64:
        return False
    expected = base64.b64encode(
        hmac.new(WEBHOOK_SECRET.encode(), raw_body, hashlib.sha256).digest()
    ).decode()
    # constant-time compare — 不可用 `==`
    return hmac.compare_digest(expected, signature_b64)


@bp.route("/api/webhooks/recur", methods=["POST"])
def recur_webhook():
    # FAIL-CLOSED: secret 未設 → 直接 503，不要 fallback skip
    if not WEBHOOK_SECRET:
        logging.error("[recur-webhook] WEBHOOK_SECRET not configured")
        return jsonify({"error": "webhook_not_configured"}), 503

    raw = request.get_data()  # 注意：必須拿 raw bytes，不是 parsed json
    signature = request.headers.get(SIGNATURE_HEADER, "")

    if not _verify_signature(raw, signature):
        logging.warning("[recur-webhook] signature mismatch")
        return jsonify({"error": "invalid_signature"}), 401

    try:
        event = json.loads(raw.decode())
    except Exception:
        return jsonify({"error": "invalid_json"}), 400

    event_id = event.get("id")
    event_type = event.get("type", "")
    if not event_id:
        return jsonify({"error": "no_event_id"}), 400

    # DEDUP: 先寫 processed_events，撞 unique → 已處理過，回 200
    try:
        col_processed_events.insert_one({
            "event_id": event_id,
            "first_seen_at": datetime.now(timezone.utc),
        })
    except DuplicateKeyError:
        logging.info("[recur-webhook] dup event_id=%s skip", event_id)
        return jsonify({"ok": True, "dedup": True}), 200

    # AUDIT LOG: 先存 raw payload
    col_recur_events.insert_one({
        "event_id": event_id,
        "ts": datetime.now(timezone.utc),
        "type": event_type,
        "payload": event,
    })

    # DISPATCH: 失敗就 500 讓 Recur 重試
    try:
        _dispatch(event)
    except Exception as e:
        logging.exception("[recur-webhook] dispatch failed event_id=%s", event_id)
        # 重要：把 processed_events 那筆 rollback，否則重試也會被當 dup
        col_processed_events.delete_one({"event_id": event_id})
        return jsonify({"error": "dispatch_failed", "detail": str(e)[:200]}), 500

    return jsonify({"ok": True}), 200


def _dispatch(event):
    event_type = event.get("type", "")
    data = event.get("data", {})

    if event_type in ("subscription.created", "subscription.updated"):
        _handle_subscription(event, data)
    elif event_type in ("subscription.deleted", "subscription.cancelled"):
        _handle_subscription_cancel(event, data)
    elif event_type in ("invoice.paid", "order.completed", "payment.succeeded"):
        _handle_payment(event, data)
    elif event_type in ("refund.created", "refund.succeeded"):
        _handle_refund(event, data)
    else:
        logging.info("[recur-webhook] no handler for %s", event_type)


def _resolve_user(event):
    """Customer_id-first lookup（P0 blocker）。
    防 email 改變後 entitlement 失聯。"""
    data = event.get("data", {})
    customer = data.get("customer", {})
    customer_id = customer.get("id")

    # 1) 優先 customer_id
    if customer_id:
        u = col_users.find_one({"recur_customer_id": customer_id})
        if u:
            return u

    # 2) fallback email + 寫回 customer_id（自動修正）
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


def _handle_subscription(event, data):
    u = _resolve_user(event)
    if not u:
        # 重要：找不到使用者不能無聲忽略，要 log 起來追
        logging.warning("[recur-webhook] sub event no user customer_id=%s",
                        data.get("customer", {}).get("id"))
        return

    # DELETION BARRIER: 帳號刪除中不接 entitlement 更新
    if u.get("deletion_started_at"):
        logging.info("[recur-webhook] skip sub for deleting user %s", u["_id"])
        return

    sub = data.get("subscription", data)  # Recur 不同版本 payload 形狀差異
    col_users.update_one(
        {"_id": u["_id"]},
        {"$set": {
            "recur_subscription_id": sub.get("id"),
            "subscription_status": sub.get("status"),
            "subscription_period_end": sub.get("current_period_end"),
            "subscription_product_id": sub.get("product_id"),
        }},
    )


def _handle_payment(event, data):
    u = _resolve_user(event)
    if not u or u.get("deletion_started_at"):
        return

    # 入點：以 product 對應的點數來算
    product_id = data.get("product_id") or data.get("line_items", [{}])[0].get("product_id")
    credits = PRODUCT_CREDIT_MAP.get(product_id, 0)
    if credits > 0:
        col_users.update_one(
            {"_id": u["_id"], "deletion_started_at": {"$exists": False}},
            {"$inc": {"credits": credits}},
        )
        col_credits_ledger.insert_one({
            "user_id": str(u["_id"]),
            "ts": datetime.now(timezone.utc),
            "delta": credits,
            "reason": "purchase",
            "event_id": event.get("id"),
            "order_id": data.get("order_id") or data.get("id"),
        })


def _handle_refund(event, data):
    """P0 blocker — refund 必須有可追溯 row"""
    u = _resolve_user(event)
    col_refund_ledger.insert_one({
        "event_id": event.get("id"),
        "ts": datetime.now(timezone.utc),
        "amount": data.get("amount"),
        "currency": data.get("currency", "TWD"),
        "user_id": str(u["_id"]) if u else None,
        "customer_id": data.get("customer", {}).get("id"),
        "original_order_id": data.get("order_id") or data.get("original_order_id"),
        "status": event.get("type"),
        "raw": event,
    })

    # 退款扣回案點（防套利：購買→入點→退款仍可用點）
    if u and not u.get("deletion_started_at"):
        product_id = data.get("product_id")
        credits = PRODUCT_CREDIT_MAP.get(product_id, 0)
        if credits > 0:
            col_users.update_one(
                {"_id": u["_id"]},
                {"$inc": {"credits": -credits}},
            )
            col_credits_ledger.insert_one({
                "user_id": str(u["_id"]),
                "ts": datetime.now(timezone.utc),
                "delta": -credits,
                "reason": "refund",
                "event_id": event.get("id"),
            })


PRODUCT_CREDIT_MAP = {
    # 在 Recur dashboard 拿 product_id 填這裡
    # "prod_xxx": 100,
}
```

### FastAPI 變體
```python
from fastapi import APIRouter, Request, HTTPException

router = APIRouter()

@router.post("/api/webhooks/recur")
async def recur_webhook(request: Request):
    if not WEBHOOK_SECRET:
        raise HTTPException(503, "webhook_not_configured")
    raw = await request.body()
    signature = request.headers.get(SIGNATURE_HEADER, "")
    if not _verify_signature(raw, signature):
        raise HTTPException(401, "invalid_signature")
    # ...剩下邏輯一樣
```

---

## Node (Express) — 完整實作

```javascript
import crypto from 'crypto';
import express from 'express';

const router = express.Router();
const WEBHOOK_SECRET = process.env.RECUR_WEBHOOK_SECRET;
const SIGNATURE_HEADER = 'x-recur-signature';

// 關鍵：webhook route 要拿 raw body，不能讓 json middleware 先 parse
function verifySignature(rawBody, signatureB64) {
  if (!WEBHOOK_SECRET || !signatureB64) return false;
  const expected = crypto
    .createHmac('sha256', WEBHOOK_SECRET)
    .update(rawBody)
    .digest('base64');
  const a = Buffer.from(expected);
  const b = Buffer.from(signatureB64);
  if (a.length !== b.length) return false;
  return crypto.timingSafeEqual(a, b);
}

router.post(
  '/api/webhooks/recur',
  express.raw({ type: 'application/json' }),  // raw body
  async (req, res) => {
    if (!WEBHOOK_SECRET) {
      return res.status(503).json({ error: 'webhook_not_configured' });
    }

    const raw = req.body;  // Buffer
    const signature = req.headers[SIGNATURE_HEADER] || '';

    if (!verifySignature(raw, signature)) {
      return res.status(401).json({ error: 'invalid_signature' });
    }

    let event;
    try {
      event = JSON.parse(raw.toString());
    } catch {
      return res.status(400).json({ error: 'invalid_json' });
    }

    const eventId = event.id;
    if (!eventId) {
      return res.status(400).json({ error: 'no_event_id' });
    }

    // DEDUP via Mongo unique index
    try {
      await db.collection('processed_events').insertOne({
        event_id: eventId,
        first_seen_at: new Date(),
      });
    } catch (e) {
      if (e.code === 11000) {
        return res.status(200).json({ ok: true, dedup: true });
      }
      throw e;
    }

    // AUDIT
    await db.collection('recur_events').insertOne({
      event_id: eventId,
      ts: new Date(),
      type: event.type,
      payload: event,
    });

    // DISPATCH
    try {
      await dispatch(event);
    } catch (err) {
      console.error('[recur-webhook] dispatch failed', eventId, err);
      await db.collection('processed_events').deleteOne({ event_id: eventId });
      return res.status(500).json({ error: 'dispatch_failed' });
    }

    res.status(200).json({ ok: true });
  }
);

async function resolveUser(event) {
  const customer = event?.data?.customer || {};
  if (customer.id) {
    const u = await db.collection('users').findOne({
      recur_customer_id: customer.id,
    });
    if (u) return u;
  }
  const email = (customer.email || '').toLowerCase();
  if (email) {
    const u = await db.collection('users').findOne({ email });
    if (u && customer.id) {
      await db.collection('users').updateOne(
        { _id: u._id },
        { $set: { recur_customer_id: customer.id } }
      );
    }
    return u;
  }
  return null;
}

// ... handlers 跟 Python 版同樣邏輯
```

### Next.js App Router 變體
```typescript
// app/api/webhooks/recur/route.ts
import { NextRequest, NextResponse } from 'next/server';
import crypto from 'crypto';

export async function POST(req: NextRequest) {
  if (!process.env.RECUR_WEBHOOK_SECRET) {
    return NextResponse.json({ error: 'webhook_not_configured' }, { status: 503 });
  }
  const raw = await req.text();  // raw string, 不是 .json()
  const signature = req.headers.get('x-recur-signature') || '';
  if (!verifySignature(raw, signature)) {
    return NextResponse.json({ error: 'invalid_signature' }, { status: 401 });
  }
  // ...
}
```

---

## 測試 webhook

### Sandbox 測試
1. Recur dashboard → 切到 test mode（pk_test_ / sk_test_）
2. 用 Recur MCP: `mcp__recur__test_webhook` 觸發測試事件
3. 確認你的 `recur_events` collection 有寫入
4. Recur MCP: `mcp__recur__get_webhook_events` 看 delivery 狀態

### 部署後驗證（before going live）
```bash
# 在 Recur dashboard 開好 webhook URL，跑：
curl -X POST https://yourdomain.com/api/webhooks/recur \
  -H "Content-Type: application/json" \
  -H "X-Recur-Signature: invalid" \
  -d '{"id":"test","type":"ping"}'
# 預期：401 invalid_signature
```

### 上 prod 前 checklist
- [ ] `RECUR_WEBHOOK_SECRET` 已 set 在 prod env
- [ ] `processed_events.event_id` unique index 已建
- [ ] `refund_ledger.event_id` unique index 已建
- [ ] 已測過 dedup（同 event_id 送兩次 → 第二次回 dedup:true）
- [ ] 已測過 dispatch 失敗回 500（用 mock 噴 exception）
- [ ] Recur dashboard webhook secret = 你 env 的 secret

## 常見坑

| 坑 | 症狀 | 解 |
|---|---|---|
| Json middleware 先 parse 掉 raw | 簽章對不上 | Express 用 `express.raw()`、Next.js 用 `await req.text()` |
| 用 `==` 比較簽章 | 理論 timing attack 風險 | 用 `compare_digest` / `timingSafeEqual` |
| Dispatch 失敗回 200 | Recur 不重試，事件遺失 | 失敗回 500，並 rollback `processed_events` |
| Email 對應靠死 | 用戶改 email 後失聯 | customer_id-first lookup |
| Webhook 沒寫 audit log | 出包後沒辦法重播 | 先寫 `recur_events` 再 dispatch |
| Deletion barrier 沒擋 webhook | 刪帳號中還在入點 | 所有 mutation 加 `deletion_started_at: {$exists: false}` |
