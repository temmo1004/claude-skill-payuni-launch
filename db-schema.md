# DB Schema — 必備 Collections / Tables

整理所有 webhook / billing / deletion 流程需要的 DB schema。MongoDB + Postgres + Prisma 三種變體。

## Schema 總覽

```
users
  + recur_customer_id (str, indexed) — webhook lookup primary key
  + recur_subscription_id (str)
  + subscription_status (str)
  + subscription_product_id (str)
  + subscription_period_end (datetime)
  + credits (int) — 點數餘額
  + deletion_started_at (datetime, sparse) — deletion barrier

processed_events    — webhook dedup
recur_events        — webhook audit log
refund_ledger       — P0：每筆退款追溯（法定 5 年保留）
credits_ledger      — 點數異動（每筆 inc/dec 留 row）
```

⚠️ 不存在的事：信用卡 token / 完整卡號 / CVV — 全在 Recur/PAYUNi 那邊，**你絕對不碰**。

---

## MongoDB (PyMongo)

### Index 建立（startup 時跑）

```python
def ensure_indexes():
    # users — recur_customer_id 是 webhook lookup hotpath
    col_users.create_index("email", unique=True, sparse=True)
    col_users.create_index("recur_customer_id", sparse=True)
    col_users.create_index(
        "deletion_started_at",
        sparse=True,
        expireAfterSeconds=600,  # 10 min — lock 過期自動 unset
    )

    # processed_events — webhook dedup（CRITICAL）
    col_processed_events.create_index("event_id", unique=True)
    col_processed_events.create_index(
        "first_seen_at",
        expireAfterSeconds=86400 * 30,  # 30 天後自動清掉
    )

    # recur_events — audit log
    col_recur_events.create_index("event_id")
    col_recur_events.create_index("type")
    col_recur_events.create_index("ts")
    col_recur_events.create_index(
        "ts",
        expireAfterSeconds=86400 * 365,  # 一年（自己評估保留期）
    )

    # refund_ledger — P0：必須查得到
    col_refund_ledger.create_index("event_id", unique=True)
    col_refund_ledger.create_index("user_id")
    col_refund_ledger.create_index("customer_id")
    col_refund_ledger.create_index("ts")
    # ⚠️ 不要設 TTL — 法定 5 年保留

    # credits_ledger
    col_credits_ledger.create_index("user_id")
    col_credits_ledger.create_index("event_id", sparse=True)  # 對應 webhook event
    col_credits_ledger.create_index("ts")
```

### TypedDict / Pydantic 模型（型別參考）

```python
from typing import TypedDict, Optional, Literal
from datetime import datetime

class User(TypedDict, total=False):
    _id: ObjectId
    email: str
    password_hash: str
    credits: int
    # Recur 關聯
    recur_customer_id: Optional[str]
    recur_subscription_id: Optional[str]
    subscription_status: Optional[Literal[
        "active", "trialing", "past_due", "canceled", "incomplete"
    ]]
    subscription_product_id: Optional[str]
    subscription_period_end: Optional[datetime]
    # Deletion barrier
    deletion_started_at: Optional[datetime]

class ProcessedEvent(TypedDict):
    event_id: str
    first_seen_at: datetime

class RecurEvent(TypedDict):
    event_id: str
    ts: datetime
    type: str
    payload: dict

class RefundLedgerRow(TypedDict, total=False):
    event_id: str            # webhook event_id（unique）
    ts: datetime
    amount: int              # TWD cents
    currency: str
    user_id: Optional[str]   # 可能 anonymize 成 "deleted:hash"
    customer_id: Optional[str]
    original_order_id: Optional[str]
    status: str              # "refund.created" / "refund.succeeded"
    raw: dict                # 原始 webhook payload

class CreditsLedgerRow(TypedDict, total=False):
    user_id: str
    ts: datetime
    delta: int               # 正 = 入點，負 = 扣點
    reason: Literal["purchase", "refund", "api:xxx", "promo", "admin_adjust"]
    event_id: Optional[str]  # webhook event 對應（idempotency）
    order_id: Optional[str]
```

---

## PostgreSQL (raw SQL)

```sql
-- users 補欄位
ALTER TABLE users ADD COLUMN IF NOT EXISTS recur_customer_id TEXT;
ALTER TABLE users ADD COLUMN IF NOT EXISTS recur_subscription_id TEXT;
ALTER TABLE users ADD COLUMN IF NOT EXISTS subscription_status TEXT;
ALTER TABLE users ADD COLUMN IF NOT EXISTS subscription_product_id TEXT;
ALTER TABLE users ADD COLUMN IF NOT EXISTS subscription_period_end TIMESTAMPTZ;
ALTER TABLE users ADD COLUMN IF NOT EXISTS credits INTEGER NOT NULL DEFAULT 0;
ALTER TABLE users ADD COLUMN IF NOT EXISTS deletion_started_at TIMESTAMPTZ;

CREATE INDEX IF NOT EXISTS idx_users_recur_customer_id
  ON users(recur_customer_id) WHERE recur_customer_id IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_users_deletion
  ON users(deletion_started_at) WHERE deletion_started_at IS NOT NULL;

-- processed_events: webhook dedup
CREATE TABLE IF NOT EXISTS processed_events (
  event_id TEXT PRIMARY KEY,
  first_seen_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- recur_events: audit log
CREATE TABLE IF NOT EXISTS recur_events (
  id BIGSERIAL PRIMARY KEY,
  event_id TEXT NOT NULL,
  ts TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  type TEXT NOT NULL,
  payload JSONB NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_recur_events_event_id ON recur_events(event_id);
CREATE INDEX IF NOT EXISTS idx_recur_events_type ON recur_events(type);
CREATE INDEX IF NOT EXISTS idx_recur_events_ts ON recur_events(ts);

-- refund_ledger: P0 blocker
CREATE TABLE IF NOT EXISTS refund_ledger (
  id BIGSERIAL PRIMARY KEY,
  event_id TEXT NOT NULL UNIQUE,
  ts TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  amount INTEGER NOT NULL,
  currency TEXT NOT NULL DEFAULT 'TWD',
  user_id TEXT,           -- 可能 anonymize
  customer_id TEXT,
  original_order_id TEXT,
  status TEXT NOT NULL,
  raw JSONB NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_refund_ledger_user ON refund_ledger(user_id);
CREATE INDEX IF NOT EXISTS idx_refund_ledger_customer ON refund_ledger(customer_id);

-- credits_ledger
CREATE TABLE IF NOT EXISTS credits_ledger (
  id BIGSERIAL PRIMARY KEY,
  user_id TEXT NOT NULL,
  ts TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  delta INTEGER NOT NULL,
  reason TEXT NOT NULL,
  event_id TEXT,
  order_id TEXT
);
CREATE INDEX IF NOT EXISTS idx_credits_ledger_user ON credits_ledger(user_id);
CREATE INDEX IF NOT EXISTS idx_credits_ledger_event ON credits_ledger(event_id)
  WHERE event_id IS NOT NULL;

-- Postgres TTL 用 pg_cron + scheduled DELETE（不像 Mongo 內建）
-- 範例：30 天後清 processed_events
-- SELECT cron.schedule('cleanup-processed-events', '0 3 * * *',
--   $$DELETE FROM processed_events WHERE first_seen_at < NOW() - INTERVAL '30 days'$$);
```

---

## Prisma (Next.js / Node)

```prisma
model User {
  id                       String    @id @default(cuid())
  email                    String    @unique
  passwordHash             String
  credits                  Int       @default(0)
  // Recur
  recurCustomerId          String?   @unique
  recurSubscriptionId      String?
  subscriptionStatus       String?
  subscriptionProductId    String?
  subscriptionPeriodEnd    DateTime?
  // Deletion barrier
  deletionStartedAt        DateTime?

  refundLedger             RefundLedger[]
  creditsLedger            CreditsLedger[]

  @@index([recurCustomerId])
  @@index([deletionStartedAt])
}

model ProcessedEvent {
  eventId      String   @id
  firstSeenAt  DateTime @default(now())
}

model RecurEvent {
  id        Int      @id @default(autoincrement())
  eventId   String
  ts        DateTime @default(now())
  type      String
  payload   Json

  @@index([eventId])
  @@index([type])
  @@index([ts])
}

model RefundLedger {
  id                Int      @id @default(autoincrement())
  eventId           String   @unique
  ts                DateTime @default(now())
  amount            Int
  currency          String   @default("TWD")
  userId            String?
  customerId        String?
  originalOrderId   String?
  status            String
  raw               Json

  user              User?    @relation(fields: [userId], references: [id])

  @@index([userId])
  @@index([customerId])
}

model CreditsLedger {
  id        Int      @id @default(autoincrement())
  userId    String
  ts        DateTime @default(now())
  delta     Int
  reason    String
  eventId   String?
  orderId   String?

  user      User     @relation(fields: [userId], references: [id])

  @@index([userId])
  @@index([eventId])
}
```

---

## Migration Notes

### 從零起新專案
直接套上面 schema，無 migration 顧慮。

### 既有專案接 Recur
1. 加上 user 表的 `recur_customer_id` 等欄位（nullable）
2. 建 `processed_events` + `recur_events` + `refund_ledger` + `credits_ledger` 4 個表
3. 既有用戶**沒**有 `recur_customer_id`，第一次付款後 webhook 用 customer_id-first lookup 自動寫入
4. 已扣過費的歷史用戶要不要 backfill？通常不用 — webhook fallback email lookup 會自動 heal

### 已有自己 credit 系統的專案
- 保留你的 ledger，加 `event_id` 欄位用來對應 Recur webhook
- `event_id` unique constraint 是 idempotency 的關鍵（重送 webhook 不會重複入點）

---

## DB 驗證 SQL（送審前自己跑）

```sql
-- 1. 每筆退款都有 row 嗎？
SELECT COUNT(*) FROM refund_ledger;  -- 應該 ≥ Recur dashboard 顯示的退款數

-- 2. 有沒有沒對應到使用者的 webhook 事件？
SELECT type, COUNT(*) FROM recur_events
  WHERE NOT EXISTS (
    SELECT 1 FROM users
    WHERE users.recur_customer_id = recur_events.payload->'data'->'customer'->>'id'
  )
  GROUP BY type;

-- 3. Dedup 是否有效？同 event_id 重複處理？
SELECT event_id, COUNT(*) FROM recur_events GROUP BY event_id HAVING COUNT(*) > 1;
-- 應該回 0 row

-- 4. 有沒有 user 卡在 deletion 中超過 10 分鐘？
SELECT id, deletion_started_at FROM users
  WHERE deletion_started_at < NOW() - INTERVAL '10 minutes';
-- → 加 cron 清這些 stale lock
```

## 常見坑

| 坑 | 症狀 | 解 |
|---|---|---|
| `email` index 不是 unique | 兩個用戶搶 webhook | unique + sparse |
| `event_id` 沒 unique | dedup 失效，webhook 重複入點 | unique index |
| `recur_customer_id` 沒 index | webhook lookup 慢 | 加 index |
| Slug / public URL 欄位沒 unique | 撞名 | unique + sparse |
| Refund ledger 設了 TTL | 法定保留期不夠 | 不設 TTL，或設 ≥ 5 年 |
| Deletion barrier TTL 設太長 | 取消失敗的用戶永遠卡 | 600 秒（10 分鐘） |
