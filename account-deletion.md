# Account Deletion + Deletion Barrier

PAYUNi P0 blocker：使用者要能**自助刪除帳號**，且刪除過程不能漏 row、不能在刪到一半時被 webhook 寫入新資料。

這檔講「deletion barrier」這個 pattern：刪除進行中設旗標，所有 mutation 入口都 check 這個旗標 → fail-closed。

## 為什麼是 P0
個資法第 11 條：使用者有權刪除其個資。PAYUNi 審查員會在 `/billing` 找「刪除帳號」按鈕，找不到直接退件。

## 整體架構

```
[User] → POST /api/account/delete (密碼確認)
              ↓
       acquire deletion lock (10 min TTL)
              ↓
       set deletion_started_at  ← BARRIER 開始
              ↓
       cancel Recur subscription (API call)
              ↓                        ↓ fail
       pre-sweep secondary collections │
              ↓                        │
       anonymize credits_ledger        │ release lock
              ↓                        │ return 502
       DELETE users row  ←─ door closes
              ↓
       second-pass sweep (清掉 race window 內的 row)
              ↓
       post-anonymize credits_ledger
              ↓
       return 200
```

## 關鍵 invariants

1. **Lock 不會被搶**：用 `findOneAndUpdate` with CAS（compare-and-set）拿 lock，撞到 → 409 already_in_progress
2. **訂閱必須先取消才刪**：避免本地清掉但 Recur 仍扣款
3. **Barrier 在 user doc 上**：所有 mutation 加 filter `deletion_started_at: {$exists: false}`
4. **Double-sweep**：刪 user doc 前後各掃一次 secondary collections，捕捉 race window 內寫入的 row
5. **Refund ledger 不刪只 anonymize**：法定 5 年保留，但 PII 要去識別化（hash uid）
6. **Token / session 都要清**：本地 session file、Redis cache、JWT blacklist

---

## Python (Flask) 完整實作

```python
import hmac, hashlib, os
from datetime import datetime, timezone
from bson import ObjectId
from flask import request, jsonify, session

# rate limit (額外保險，防暴力刪除)
@bp.route("/api/account/delete", methods=["POST"])
@login_required
@rate_limit(max_per_min=3)  # 你的 rate limit decorator
def delete_account():
    uid = _current_user_id()
    data = request.get_json() or {}
    password = data.get("password", "")

    # 1. 密碼確認（reauth）
    user = col_users.find_one({"_id": ObjectId(uid)})
    if not user:
        return jsonify({"ok": False, "error": "user_not_found"}), 404
    if not _verify_password(password, user.get("password_hash")):
        return jsonify({"ok": False, "error": "invalid_password"}), 401

    # 2. 嘗試拿 deletion lock（CAS）
    lock_result = col_users.update_one(
        {
            "_id": ObjectId(uid),
            "deletion_started_at": {"$exists": False},
        },
        {"$set": {"deletion_started_at": datetime.now(timezone.utc)}},
    )
    if lock_result.matched_count == 0:
        return jsonify({"ok": False, "error": "deletion_already_in_progress"}), 409

    # 3. 重讀 user（拿 lock 後可能 sub_id 被 webhook 改了）
    user = col_users.find_one({"_id": ObjectId(uid)}) or user

    # 4. 取消 Recur 訂閱（如有）— 必須先取消才能刪
    sub_id = user.get("recur_subscription_id")
    sub_status = user.get("subscription_status")
    if sub_id and sub_status in ("active", "trialing", "past_due"):
        try:
            import requests
            r = requests.delete(
                f"{RECUR_API_BASE}/v1/subscriptions/{sub_id}",
                headers={"Authorization": f"Bearer {RECUR_SECRET_KEY}"},
                timeout=10,
            )
            if r.status_code >= 400:
                # 釋放 lock，讓使用者改去 Customer Portal 手動取消
                col_users.update_one(
                    {"_id": ObjectId(uid)},
                    {"$unset": {"deletion_started_at": ""}},
                )
                return jsonify({
                    "ok": False,
                    "error": "subscription_cancel_failed",
                    "hint": "請先到付款管理頁取消訂閱後再刪帳號",
                }), 502
        except Exception:
            col_users.update_one(
                {"_id": ObjectId(uid)},
                {"$unset": {"deletion_started_at": ""}},
            )
            return jsonify({"ok": False, "error": "subscription_cancel_failed"}), 502

    # 5. 建 anonymize marker (HMAC for non-deterministic across envs)
    secret = (app.secret_key or "fallback").encode()
    deleted_marker = "deleted:" + hmac.new(
        secret, uid.encode(), hashlib.sha256
    ).hexdigest()[:32]

    # 6. 列出所有 per-user collections（之後要全清）
    per_user_collections = [
        col_profiles, col_messages, col_sent, col_copies,
        col_posts, col_bookings, col_tokens, col_sessions,
        col_schedules, col_api_usage,
        # ⚠️ 不要加 col_refund_ledger（要保留法定 5 年）
        # ⚠️ 不要加 col_credits_ledger（要 anonymize 不是刪）
    ]

    # 7. Pre-sweep — fail-closed：任一 collection 刪除失敗 → ABORT
    wipe_failed_at = None
    for coll in per_user_collections:
        try:
            coll.delete_many({"user_id": uid})
        except Exception as e:
            wipe_failed_at = getattr(coll, "name", "?")
            break
    if wipe_failed_at:
        # 重要：不刪 user doc，lock 保留讓 retry / admin 處理
        return jsonify({
            "ok": False,
            "error": "wipe_failed",
            "stage": wipe_failed_at,
        }), 503

    # 8. Pre-anonymize credits_ledger（PII 去識別化）
    col_credits_ledger.update_many(
        {"user_id": uid},
        {"$set": {"user_id": deleted_marker}},
    )

    # 9. Delete user doc ← DOOR CLOSES
    # 任何 mutation 加 deletion_started_at filter 後，這之後就 fail
    col_users.delete_one({"_id": ObjectId(uid)})

    # 10. Post-anonymize（race window 內可能有新 ledger row）
    col_credits_ledger.update_many(
        {"user_id": uid},
        {"$set": {"user_id": deleted_marker}},
    )

    # 11. Second-pass sweep — race window 內寫入的 row 清掉
    for coll in per_user_collections:
        try:
            coll.delete_many({"user_id": uid})
        except Exception:
            pass  # best-effort，admin sweep cron 之後再清

    # 12. 清 token / session 檔案 / Redis cache
    _cleanup_session_data(uid)

    # 13. Refund ledger anonymize（不刪）
    col_refund_ledger.update_many(
        {"user_id": uid},
        {"$set": {"user_id": deleted_marker}},
    )

    session.clear()
    return jsonify({"ok": True})
```

## Deletion Barrier — 怎麼在 mutation 入口加

**規則**：所有會寫 user-scoped 資料的端點 + webhook handler，都要加這個 barrier。

### 1. login_required decorator 加 barrier
```python
def login_required(f):
    @wraps(f)
    def wrapper(*args, **kwargs):
        uid = session.get("user_id")
        if not uid:
            return jsonify({"error": "unauthorized"}), 401
        # BARRIER: 刪除進行中 → 拒絕
        user = col_users.find_one(
            {"_id": ObjectId(uid)},
            {"deletion_started_at": 1},
        )
        if not user:
            session.clear()
            return jsonify({"error": "user_deleted"}), 401
        if user.get("deletion_started_at"):
            return jsonify({"error": "deletion_in_progress"}), 423
        return f(*args, **kwargs)
    return wrapper
```

### 2. Webhook handler 加 barrier（最容易被忽略）
```python
def _handle_payment(event, data):
    u = _resolve_user(event)
    if not u:
        return
    # BARRIER: 不要在刪除中入點
    if u.get("deletion_started_at"):
        logging.info("[webhook] skip payment for deleting uid=%s", u["_id"])
        return
    # ... 入點邏輯
```

### 3. Credits update 加 atomic barrier
```python
# 用 conditional update，撞到 deletion_started_at → matched=0
result = col_users.update_one(
    {
        "_id": ObjectId(uid),
        "deletion_started_at": {"$exists": False},  # ← barrier
    },
    {"$inc": {"credits": delta}},
)
if result.matched_count == 0:
    return jsonify({"error": "account_deleting"}), 423
```

### 4. Session / cache 也要清
```python
def _cleanup_session_data(uid):
    # Redis session
    redis_client.delete(f"session:{uid}:*")
    # Token files
    token_path = os.path.join(TOKEN_DIR, f"{uid}.txt")
    try:
        os.unlink(token_path)
    except FileNotFoundError:
        pass
    # JWT blacklist (如果用 JWT)
    blacklist_user_tokens(uid)
```

---

## Node (Express) 變體

```javascript
router.post('/api/account/delete', requireAuth, async (req, res) => {
  const uid = req.session.userId;
  const { password } = req.body;

  // 1. Reauth
  const user = await db.collection('users').findOne({ _id: new ObjectId(uid) });
  if (!user || !await verifyPassword(password, user.password_hash)) {
    return res.status(401).json({ error: 'invalid_password' });
  }

  // 2. CAS lock
  const lockResult = await db.collection('users').updateOne(
    { _id: new ObjectId(uid), deletion_started_at: { $exists: false } },
    { $set: { deletion_started_at: new Date() } }
  );
  if (lockResult.matchedCount === 0) {
    return res.status(409).json({ error: 'deletion_already_in_progress' });
  }

  // ...剩下流程同 Python 版
});
```

---

## UI — `/billing` 頁底刪帳號按鈕

```html
<section class="border-t border-red-200 pt-6 mt-12">
  <h3 class="text-red-700 font-bold mb-2">危險區域</h3>
  <p class="text-sm text-gray-600 mb-3">
    刪除帳號將永久清除您的所有資料。訂閱會自動取消，但已支付的款項依《退款政策》辦理。
  </p>
  <button id="delete-account-btn" class="bg-red-600 text-white px-4 py-2 rounded">
    刪除帳號
  </button>
</section>

<script>
document.getElementById('delete-account-btn').addEventListener('click', async () => {
  const pw = prompt('請輸入密碼確認刪除（此動作無法復原）');
  if (!pw) return;
  if (!confirm('真的要刪除帳號？所有資料會永久消失。')) return;
  const r = await fetch('/api/account/delete', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ password: pw }),
  });
  const d = await r.json();
  if (d.ok) {
    alert('帳號已刪除');
    location.href = '/';
  } else if (d.error === 'subscription_cancel_failed') {
    alert(d.hint || '請先到付款管理頁取消訂閱');
  } else if (d.error === 'invalid_password') {
    alert('密碼錯誤');
  } else {
    alert(d.error || '刪除失敗，請稍後再試');
  }
});
</script>
```

---

## 測試 deletion barrier

### 單元測試重點
1. **CAS lock**：兩個並發 request 進 delete，只有一個成功，另一個 409
2. **訂閱取消失敗 → lock 釋放**：mock Recur API 回 500，確認 `deletion_started_at` 被 $unset
3. **Pre-sweep 失敗 → ABORT**：mock 某 collection raise exception，確認 user doc 沒被刪
4. **Race window**：刪除中送 payment.succeeded webhook，確認被 barrier 擋掉

### 過審 demo 流程
1. 在 prod 註冊一個 demo account
2. 訂閱（讓有 Recur subscription）
3. 在 `/billing` 點刪帳號
4. 完整跑完一次給審查員看 — 重點是「能找到按鈕」+「能刪掉」

## 常見坑

| 坑 | 症狀 | 解 |
|---|---|---|
| 沒先取消訂閱就刪 user | PAYUNi 還在扣，客訴爆 | 第 4 步硬性 cancel API call |
| 只刪 user doc，secondary collections 不管 | DB 一堆 orphan row | per_user_collections 全列出 |
| 把 refund_ledger 也刪了 | 違反法定保留 | refund 用 anonymize 不用 delete |
| Barrier 只加在 login_required | Webhook 寫入仍會發生 | webhook handler 也要查 deletion_started_at |
| Lock 沒 TTL | 取消失敗後永遠卡住 | 加 cron 清 deletion_started_at > 10 min 的 row |
| Token file 沒清 | 重綁時可能 resurrect | unlink token file + clear Redis session |
