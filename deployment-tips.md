# 部署 Tips — Env Vars / Sandbox→Live Cutover / Email Infra

## 必備環境變數清單

```bash
# Recur / PAYUNi（必填）
RECUR_PUBLISHABLE_KEY=pk_live_xxx   # 前端可見
RECUR_SECRET_KEY=sk_live_xxx        # 後端 only，絕不暴露
RECUR_WEBHOOK_SECRET=whsec_xxx      # webhook 簽章驗證
RECUR_API_BASE=https://api.recur.tw # sandbox 可能不同 host

# 應用層
APP_BASE_URL=https://yourdomain.com  # 用於 success_url / return_url 組裝
SECRET_KEY=<32+ chars random>        # session cookie / HMAC marker

# DB
MONGODB_URI=mongodb+srv://...
# 或
DATABASE_URL=postgres://...

# Email（自有域名收信 + 寄信）
SMTP_HOST=smtp.zsend.zeabur.com  # 或 SendGrid / Mailgun
SMTP_USER=apikey
SMTP_PASS=<api key>
SUPPORT_EMAIL=support@yourdomain.com

# 業務 config
ALLOWED_PRODUCT_IDS=prod_basic,prod_pro
PRODUCT_CREDIT_MAP={"prod_basic":100,"prod_pro":300}  # JSON 字串
```

## Test → Live Cutover Checklist

⚠️ **常見坑**：UI 顯示「金流上了」但 env 還是 `pk_test_` / `sk_test_` — 用戶真的去刷卡會失敗。

```bash
# 換 key 前確認
[ ] Recur dashboard 已切到 Live mode
[ ] 已在 dashboard 拿新的 publishable_key (pk_live_*) / secret_key (sk_live_*)
[ ] Live 模式的 webhook endpoint 已建（test / live 是分開的 webhook config）
[ ] 拿到 live mode 的 webhook secret（whsec_live_*）

# 更新 env
[ ] RECUR_PUBLISHABLE_KEY = pk_live_*
[ ] RECUR_SECRET_KEY = sk_live_*
[ ] RECUR_WEBHOOK_SECRET = whsec_live_*

# 部署
[ ] 重啟服務（env var 不會 hot reload）
[ ] runtime log 確認沒因 key 切換爆炸

# 端到端驗證
[ ] 自己刷一張最便宜方案（NT$1 也行）
[ ] DB 的 recur_events 收到 live event
[ ] processed_events 寫入 event_id
[ ] credits_ledger 入點
[ ] /billing 顯示新訂閱狀態
```

### Startup guard（防忘記換）

```python
# app.py 啟動時 sanity check
import os, sys, logging

pk = os.environ.get("RECUR_PUBLISHABLE_KEY", "")
sk = os.environ.get("RECUR_SECRET_KEY", "")
env_name = os.environ.get("ENV", "production")

if env_name == "production":
    if pk.startswith("pk_test_") or sk.startswith("sk_test_"):
        logging.critical("[STARTUP] test keys deployed to production!")
        sys.exit(1)
```

## Email Infra（自有域名）

**PAYUNi MEDIUM finding**：客服 email 用 gmail / hotmail → 「不符合企業形象」要求改。

### 推薦組合（成本低）

```
你的域名 yourdomain.com
  ├─ MX → Cloudflare Email Routing（免費）
  │       └─ 轉發 support@yourdomain.com → 你的 gmail
  └─ SPF / DKIM → Zeabur ZSend or SendGrid
          └─ 用來寄系統信（驗證信、收據、退款通知）
```

### Cloudflare Email Routing 設定
1. CF dashboard → 你的域名 → Email → Email Routing
2. 啟用 → 自動建 MX records
3. Routes → `support@yourdomain.com` → `you@gmail.com`（轉發）
4. （可選）catch-all → `*@yourdomain.com` → gmail

### ZSend（Zeabur）設定
1. Zeabur dashboard → 任意 project → Add Service → ZSend
2. 拿 API key
3. CF DNS 加 SPF / DKIM record（ZSend dashboard 會給）
4. 發送方填 `noreply@yourdomain.com`

### policy 頁聯絡資訊範本
```
客服信箱：support@yourdomain.com   ← Cloudflare routing 接收
帳務問題：billing@yourdomain.com
資料權利：privacy@yourdomain.com
```

(這 3 個都可以全 route 到同一個 gmail，但**對外顯示用域名 email**)

## 各平台部署備忘

### Zeabur
```bash
# 看 service runtime log
npx zeabur@latest deployment log --service-id <id> -t runtime -i=false

# 設環境變數
npx zeabur@latest variable update --service-id <id> \
  --key RECUR_WEBHOOK_SECRET --value <secret> -i=false

# 重啟（env var 不會 hot reload）
npx zeabur@latest service restart --service-id <id> -i=false
```

### Vercel
- Webhook endpoint：用 App Router 的 `app/api/webhooks/recur/route.ts`
- ⚠️ Edge runtime 沒 Node crypto，要明確 `export const runtime = 'nodejs'`
- 環境變數在 dashboard 設，記得 separate prod / preview

### Railway
- 直接設 Variables → Deploy
- Webhook：service network → public network → 拿 URL 填 Recur dashboard

## Webhook URL 註冊清單

部署完成後到 Recur dashboard 設好 webhook：

```
URL:    https://yourdomain.com/api/webhooks/recur
Events: subscription.created
        subscription.updated
        subscription.deleted
        invoice.paid
        order.completed
        payment.succeeded
        refund.created
        refund.succeeded
Secret: whsec_xxx（拿來填到 env RECUR_WEBHOOK_SECRET）
```

## 部署後 smoke test

```bash
# 1. health check
curl https://yourdomain.com/healthz

# 2. webhook 驗章 fail-closed 測試
curl -X POST https://yourdomain.com/api/webhooks/recur \
  -H "Content-Type: application/json" \
  -H "X-Recur-Signature: invalid" \
  -d '{"id":"smoke","type":"ping"}'
# 預期：401

# 3. webhook secret 沒設時 503
# (先把 env 取消設定再測，或在 staging 環境測)

# 4. policy 頁全部 200
for path in /policy/privacy /policy/terms /policy/refund /policy/payment-info /policy/contact; do
  curl -o /dev/null -s -w "%{http_code} $path\n" https://yourdomain.com$path
done
# 全部 200

# 5. 英文 mirror 也要 200
for path in /en/policy/privacy /en/policy/terms /en/policy/refund /en/policy/payment-info /en/policy/contact /en/billing; do
  curl -o /dev/null -s -w "%{http_code} $path\n" https://yourdomain.com$path
done
```

## Recur MCP 工具（用於 Claude Code 中）

裝 Recur MCP 後可在 Claude Code 內直接操作 Recur：

```bash
claude mcp add --transport http --scope user recur https://mcp.recur.tw/
# 跑 OAuth 流程
```

常用工具：
- `mcp__recur__create_api_key` — 建 prod key
- `mcp__recur__create_webhook` — 建 webhook config
- `mcp__recur__list_products` — 看 products
- `mcp__recur__get_production_readiness_checklist` — **送審前必跑**
- `mcp__recur__test_webhook` — 觸發測試事件
- `mcp__recur__get_recent_api_calls` — debug webhook delivery
- `mcp__recur__run_health_check` — 整體健康度

送審前**必跑**：
```
mcp__recur__get_production_readiness_checklist
# 應該 ≥ 95% pass
```

## 常見坑

| 坑 | 症狀 | 解 |
|---|---|---|
| Test key 上 prod | 真實付款 fail | startup guard 擋 `pk_test_` |
| 換 key 沒重啟 | 還是讀舊 env | env var 改完 → restart |
| Live webhook secret ≠ test secret | 簽章對不上 | 兩個 mode 的 secret 不同，要分開設 |
| Webhook URL 還指向 staging | prod 拿不到事件 | dashboard 改 endpoint URL |
| Email 寄信 from gmail | 進垃圾桶 | 用自有域名 + SPF/DKIM |
| Edge runtime 用 crypto | webhook 簽章驗證 crash | `export const runtime = 'nodejs'` |
