# Idempotency & Race Condition Patterns

PAYUNi audit R9B01 抓到 **Recur webhook retry double-grant** — 同一個 payment event 重試兩次導致用戶被入點兩次。這個檔講送審必備的 idempotency / atomic / race-condition patterns。

「Dedup table 把同 event_id 擋掉」**不夠**。Recur webhook 有 retry policy，會在 4xx/5xx 後重送；如果你 dispatch 一半 crash，dedup row 已寫但入點沒做完，重試會被 dedup 擋下變成「永遠 grant 不到」— **這才是真實 race**。

---

## Pattern 1：Idempotency Lease（send / scrape 雙擊保護）

避免使用者雙擊送出 / 30 秒內重複請求。

```python
import hashlib
from datetime import datetime, timezone, timedelta

LEASE_TTL_SEC = 5  # 短 TTL（不要 60 — UX 太差）

def _send_lease_key(uid: str, target: str, body: str) -> str:
    """Hash body 避免長 key — 同 body 才視為 dup"""
    h = hashlib.sha256(body.encode()).hexdigest()[:16]
    return f"send:{uid}:{target}:{h}"


def claim_send_lease(uid, target, body):
    """Returns (claimed: bool, retry_in: int)"""
    key = _send_lease_key(uid, target, body)
    now = datetime.now(timezone.utc)
    expires = now + timedelta(seconds=LEASE_TTL_SEC)
    try:
        col_send_leases.insert_one({
            "_id": key,
            "uid": uid,
            "expires_at": expires,
            "created_at": now,
        })
        return True, 0
    except DuplicateKeyError:
        # 算還剩幾秒
        existing = col_send_leases.find_one({"_id": key})
        if existing:
            remain = (existing["expires_at"] - now).total_seconds()
            return False, max(1, int(remain))
        return False, LEASE_TTL_SEC


def release_send_lease(uid, target, body):
    """失敗時主動釋放 — 否則用戶 5 秒內無法重試"""
    col_send_leases.delete_one({"_id": _send_lease_key(uid, target, body)})


# Index TTL 自動清過期 lease
col_send_leases.create_index("expires_at", expireAfterSeconds=0)
```

**關鍵 invariants**：
- 失敗就 `release` — 否則卡 TTL 期間，用戶體驗爛
- 成功不釋放 — 留著擋雙擊
- TTL 短（5s）— 太長 = 改文字 retry 也被擋
- Index TTL 自動清 — 不要每天跑 cron 清

**前端配合**：
```javascript
const r = await fetch('/api/send', {...});
const d = await r.json();
if (d.code === 'duplicate_send') {
  // silent skip 不彈 alert（dup 不是錯誤）
  return;
}
```

---

## Pattern 2：Recur Webhook Retry-Safe Dispatch

R9B01 finding：「webhook dispatch 失敗時，dedup row 已寫，retry 變成 no-op，credits 永遠不到」。

**錯誤版**（最早 webhook 實作）：
```python
try:
    col_processed_events.insert_one({"event_id": event_id})  # dedup ← 已寫
    dispatch(event)  # ← crash here
except DuplicateKeyError:
    return 200  # ← 重試也被 dedup 擋掉，credits 永遠不到
```

**正確版**：dispatch 失敗就 rollback dedup
```python
try:
    col_processed_events.insert_one({"event_id": event_id})
except DuplicateKeyError:
    return jsonify({"ok": True, "dedup": True}), 200

try:
    dispatch(event)
except Exception:
    # ★ 關鍵：rollback dedup row 讓 retry 能重跑
    col_processed_events.delete_one({"event_id": event_id})
    return jsonify({"error": "dispatch_failed"}), 500
```

**更穩**：每個 dispatch 子操作自帶 `event_id` idempotency anchor
```python
def _handle_payment(event, data):
    # 入點時用 event_id 做 atomic check
    result = col_credits_ledger.update_one(
        {"event_id": event["id"]},  # ← 已存在 = 已入過
        {"$setOnInsert": {
            "user_id": str(u["_id"]),
            "event_id": event["id"],
            "delta": credits,
            "reason": "purchase",
            "ts": datetime.now(timezone.utc),
        }},
        upsert=True,
    )
    if result.upserted_id is None:
        # 已入過 — skip，但仍要算成功
        logging.info("[payment] already credited event=%s", event["id"])
        return

    # 真的是新事件才 increment user credits
    col_users.update_one(
        {"_id": u["_id"], "deletion_started_at": {"$exists": False}},
        {"$inc": {"credits": credits}},
    )
```

→ 即使 dedup table 不存在，每個 ledger row 都有 unique `event_id`，**重入點不會發生**。

---

## Pattern 3：CAS（Compare-And-Set）modified_count 檢查

R9B08 finding：「`update_one` matched_count 包含同值更新，不等於真正改了狀態」。

**錯誤版**：
```python
result = col_users.update_one(
    {"_id": uid, "status": "pending"},
    {"$set": {"status": "active"}},
)
if result.matched_count == 0:
    return "not pending"
# 問題：matched_count 算的是 filter 命中，不是「真的改了」
```

**正確版**：用 `modified_count`
```python
result = col_users.update_one(
    {"_id": uid, "status": "pending"},
    {"$set": {"status": "active"}},
)
if result.modified_count == 0:
    # 真的沒改 — 可能 race 中已被別人改成 active 或其他狀態
    return "no_change"
```

**用 CAS 拿 deletion lock**（這個只看 matched 才對 — 因為要設新欄位）：
```python
result = col_users.update_one(
    {"_id": uid, "deletion_started_at": {"$exists": False}},
    {"$set": {"deletion_started_at": datetime.now(timezone.utc)}},
)
if result.matched_count == 0:
    return "already_in_progress"
# 這裡用 matched_count 是對的 — 因為 deletion_started_at 設定前不存在
```

---

## Pattern 4：Deletion Barrier 全鏈路

`account-deletion.md` 講主流程，這檔補**所有需要加 barrier 的地方**。R4B01-R4B07 補了 22 個 race window。

| 入口 | Race 場景 | Barrier 寫法 |
|---|---|---|
| `login_required` decorator | 刪除中 session 仍 valid | `if user.deletion_started_at: return 423` |
| Webhook handler | webhook 在刪除中入點 | `if u.get("deletion_started_at"): return` |
| Credits inc | 刪除中還在 paywall 扣點 | filter `deletion_started_at: {$exists: False}` |
| Scheduler ghost-send | 刪除前排程的 send 仍會執行 | scheduler 啟動前先 query user |
| AI draft atomic claim | 刪除中還寫 draft | upsert filter 加 barrier |
| Public booking endpoint | 公開頁 booking 沒驗 user | landing 頁先 fail-closed query user |
| Token file disk | 刪 DB 但 disk token 還在 | account-deletion `os.unlink` token file |
| Bridge webhook | bridge 寫 user-scoped 資料 | bridge 端也要查 user 狀態 |
| Scrape quota release | 刪除中還在累加額度 | quota update 加 barrier |
| Tag assign re-check | tag 批次更新跨多 user | 每筆 atomic 帶 barrier |
| Register zombie recycle | 刪到一半被同 email 註冊 | unique email + deleted_marker hash |

**驗收**：grep 所有寫 user-scoped DB 的地方，看有沒有 `deletion_started_at` filter。

---

## Pattern 5：Atomic Quota Decrement（防超用）

ba6b286 免費額度系統 5 次/月 — 防併發 race 超送。

```python
# 錯誤版：先 read 後 write
quota = col_users.find_one({"_id": uid})["quota_remaining"]
if quota <= 0:
    return 402
do_action()
col_users.update_one({"_id": uid}, {"$inc": {"quota_remaining": -1}})
# 兩個 concurrent request 都 read 到 1，都做了，結果 -1

# 正確版：atomic conditional decrement
result = col_users.update_one(
    {
        "_id": ObjectId(uid),
        "quota_remaining": {"$gte": 1},
        "deletion_started_at": {"$exists": False},
    },
    {
        "$inc": {"quota_remaining": -1},
        "$set": {"quota_period_anchor": current_month_anchor()},
    },
)
if result.matched_count == 0:
    return jsonify({"error": "quota_exceeded"}), 402
do_action()
```

**月度重置**：用 `quota_period_anchor` 比對當月，過月自動重置 — 不要靠 cron 跑「每月一號重置」會踩時區雷。

```python
def current_month_anchor():
    """yyyy-mm format using Asia/Taipei timezone"""
    from zoneinfo import ZoneInfo
    now = datetime.now(ZoneInfo("Asia/Taipei"))
    return f"{now.year:04d}-{now.month:02d}"


def get_quota_remaining(user):
    if user.get("quota_period_anchor") != current_month_anchor():
        # 過月 — 自動重置
        col_users.update_one(
            {"_id": user["_id"]},
            {"$set": {
                "quota_remaining": QUOTA_MONTHLY,
                "quota_period_anchor": current_month_anchor(),
            }},
        )
        return QUOTA_MONTHLY
    return user.get("quota_remaining", QUOTA_MONTHLY)
```

---

## Pattern 6：Promo / 兌換碼 vs 訂閱 Union Check（R6）

a892ae3 finding：「promo 給 100 點，但用戶剛訂閱也給 100 點 — 套利攻擊」。

**正確版**：promo redeem 時，check 是否已透過訂閱拿到等值權益

```python
def redeem_promo(uid, code):
    promo = col_promos.find_one({"code": code})
    if not promo or promo.get("expires_at") < datetime.now(timezone.utc):
        return {"error": "invalid_code"}, 400

    # 防 double-redeem
    existing = col_promo_redemptions.find_one({
        "user_id": uid,
        "code": code,
    })
    if existing:
        return {"error": "already_redeemed"}, 409

    # ★ Union check：免費 promo 給的東西，跟訂閱 / 加購包重複嗎？
    if promo["type"] == "free_trial":
        u = col_users.find_one({"_id": ObjectId(uid)})
        if u.get("subscription_status") in ("active", "trialing"):
            return {"error": "already_has_subscription"}, 409
        if u.get("trial_used"):
            return {"error": "trial_already_used"}, 409

    # 原子操作：寫 redemption + 入點 + 標記 trial_used
    try:
        col_promo_redemptions.insert_one({
            "user_id": uid,
            "code": code,
            "redeemed_at": datetime.now(timezone.utc),
            "promo_type": promo["type"],
        })
    except DuplicateKeyError:
        return {"error": "already_redeemed"}, 409

    col_users.update_one(
        {"_id": ObjectId(uid), "deletion_started_at": {"$exists": False}},
        {
            "$inc": {"credits": promo["credits"]},
            "$set": {"trial_used": True} if promo["type"] == "free_trial" else {},
        },
    )
```

**Index**：`col_promo_redemptions.create_index([("user_id", 1), ("code", 1)], unique=True)`

---

## Pattern 7：Rollback Recovery（R6 / R9）

a892ae3 finding：「promo redemption 寫了，credits 入了，但 user 在第三步 deletion barrier 撞到 → ledger 留 orphan row」。

**正確版**：critical path 要有 rollback 機制
```python
def redeem_promo(uid, code):
    # ... 前置 check ...

    # Step 1: 標記 redemption（容易 rollback）
    redemption_id = col_promo_redemptions.insert_one({
        "user_id": uid,
        "code": code,
        "status": "pending",
        "ts": datetime.now(timezone.utc),
    }).inserted_id

    # Step 2: try inc credits
    result = col_users.update_one(
        {"_id": ObjectId(uid), "deletion_started_at": {"$exists": False}},
        {"$inc": {"credits": promo["credits"]}},
    )
    if result.matched_count == 0:
        # ROLLBACK
        col_promo_redemptions.update_one(
            {"_id": redemption_id},
            {"$set": {"status": "rollback", "reason": "user_deleting"}},
        )
        return {"error": "account_deleting"}, 423

    # Step 3: confirm redemption
    col_promo_redemptions.update_one(
        {"_id": redemption_id},
        {"$set": {"status": "redeemed"}},
    )

    # Step 4: ledger
    col_credits_ledger.insert_one({
        "user_id": uid,
        "delta": promo["credits"],
        "reason": "promo",
        "promo_redemption_id": redemption_id,
        "ts": datetime.now(timezone.utc),
    })
```

每個 critical write 留可查證的 row，PAYUNi 對帳時 trace 得回去。

---

## Pattern 8：時區一致性

R9B01 / R9B03 finding：「scheduler / 計費日期 / quota 重置 用 server local time（UTC），但用戶看到的是台北時間 — 出現「8 月 31 日下午 11 點」入點被算 9 月」。

**規則**：
1. DB 全存 **UTC**（`datetime.now(timezone.utc)`，**不要** `datetime.utcnow()` — 是 naive）
2. 用戶介面顯示 **Asia/Taipei**
3. 業務邏輯（quota 月度切點 / 計費日期）用 **Asia/Taipei**

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

TPE = ZoneInfo("Asia/Taipei")

# 寫 DB
now_utc = datetime.now(timezone.utc)

# 顯示給用戶
display = now_utc.astimezone(TPE).strftime("%Y-%m-%d %H:%M")

# 業務邏輯（月度 quota 切點）
def quota_period_anchor():
    return datetime.now(TPE).strftime("%Y-%m")
```

**Test setup**：CI 跑測試前 `TZ=UTC pytest` 抓出忘記 timezone 的 bug。

---

## 自查表

跑這 8 條過你的 codebase：

- [ ] Send / scrape 雙擊 → 有 idempotency lease？失敗有 release？
- [ ] Webhook dispatch 失敗 → rollback dedup row？
- [ ] Credits inc 用 `event_id` 做 ledger upsert？
- [ ] CAS check 用 `modified_count` 不是 `matched_count`？（除了純 set 新欄位）
- [ ] 所有 user-scoped mutation 加 `deletion_started_at` barrier？
- [ ] Quota decrement atomic conditional `$gte`？
- [ ] Promo / 兌換碼 有 union check 防套利？
- [ ] DB 用 UTC、業務邏輯用 Asia/Taipei、不用 `utcnow()`？

每條 yes 才安全。
