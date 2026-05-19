# PAYUNi 送審 Policy 頁中英版骨架

從實戰過審文案反推的 6 份 policy 頁模板。**使用前必改**：品牌名、聯絡資訊、業務範圍、AI / LINE 聲明（要跟你實際做的事一致，不能矛盾）。

PAYUNi 強制檢查項：**Refund Policy** 必有、**Payment Info** 必寫「PAYUNi 統一金流處理 + 不儲信用卡」。

---

## 1. Privacy Policy

**路徑**：`/policy/privacy` + `/en/policy/privacy`

### 中文骨架
```
# 隱私權政策

更新日期：YYYY-MM-DD

## § 1 資料蒐集範圍
我們蒐集：
- 帳號資料（email、密碼 hash）
- 付款資料（由 PAYUNi 統一金流處理，本服務不儲存信用卡）
- 服務使用紀錄（操作 log、IP、瀏覽器資訊）
- {如有接 LINE：LINE 帳號 ID、好友列表、訊息 metadata}

## § 2 資料用途
僅用於：提供本服務功能、帳務對帳、客服回應、法令遵循。

## § 3 第三方分享
僅在下列情況分享：
- 金流：PAYUNi（必要）
- 寄信：{你用的 ESP，例如 ZSend / SendGrid}
- 雲端基礎設施：{Zeabur / AWS}
- 司法機關依法調閱

## § 4 Cookie 政策
使用 session cookie 維持登入狀態，不使用第三方廣告追蹤。

## § 5 使用者權利
依個人資料保護法第 3 條，您可：查詢、複製、更正、刪除您的個人資料。
帳號刪除請於 /billing 頁底點「刪除帳號」。

## § 6 資料保留期間
帳號存續期間保留；刪除帳號後 30 天內完成清除（退款 ledger 為法定保留 5 年）。

## § 7 AI 使用聲明
{選一}
- 本服務不使用 AI 自動處理使用者資料。
- 本服務使用 LLM 生成 {功能名稱}，輸入內容僅暫存於請求處理期間，不用於模型訓練。

## § 8 {如有接 LINE} LINE 資料特別保護
本服務透過使用者授權的 LINE 桌面端與 LINE 平台互動，不儲存 LINE 對話內容明文。
LINE 訊息僅作 metadata（時間 / 對象 mid）記錄供統計，明文內容不離開使用者瀏覽器。
LINE 平台的服務品質非本服務可控；投遞依 LINE 平台狀態。

## § 9 聯絡方式
若對隱私權政策有疑問，請寄信至 support@yourdomain.com
```

### 英文骨架
```
# Privacy Policy

Last updated: YYYY-MM-DD

## § 1 Data we collect
- Account: email, hashed password
- Payment: processed by PAYUNi; we do not store credit card numbers
- Service logs: actions, IP, browser
- {If LINE: LINE account id, contact list, message metadata}

## § 2 Purpose
Service delivery, billing reconciliation, customer support, legal compliance only.

## § 3 Third-party sharing
- Payment: PAYUNi
- Email: {your ESP}
- Hosting: {your infra}
- Legal authorities upon lawful request

## § 4 Cookies
Session cookies for login state. No third-party ad trackers.

## § 5 Your rights
Per Taiwan PIPA Art. 3, you may access, copy, correct, or delete your data.
Self-service account deletion at /en/billing footer.

## § 6 Retention
While account is active; deleted within 30 days after account deletion.
Refund ledger retained 5 years per Taiwan tax law.

## § 7 AI usage
{Pick one}
- We do not use AI to process user data.
- We use LLM to generate {feature}. Inputs are not used for model training.

## § 8 {If LINE} LINE-specific data protection
We interact with LINE via user-authorized desktop client. We do not store LINE message plaintext.
Delivery depends on LINE platform availability and is not guaranteed.

## § 9 Contact
For privacy questions: support@yourdomain.com
```

---

## 2. Terms of Service

**路徑**：`/policy/terms` + `/en/policy/terms`

### 必須對齊 Privacy 的條款
- AI 使用聲明：跟 Privacy § 7 一字不差
- LINE 投遞承諾：跟 Privacy § 8 一字不差，**不寫「100% 送達」「保證即時」「永不漏發」**

### 骨架
```
# 服務條款

## § 1 服務內容
{一句話定位，例：本服務為房仲業者提供 LINE 群發與案件管理工具}

## § 2 帳號註冊
- 須滿 18 歲
- 須以真實 email 註冊
- 嚴禁多帳號規避免費額度

## § 3 付費與退款
- 訂閱依 § 4 自動續訂，可於 /billing 隨時取消
- 退款依《退款政策》辦理
- 付款由 PAYUNi 處理，本服務不儲信用卡

## § 4 訂閱與取消
- 訂閱結束日後失效，不退已使用月份
- 取消後當月仍可使用至 period_end
- 取消請至 /billing → 管理訂閱

## § 5 使用限制
- 不得用於詐騙、騷擾、垃圾訊息
- 不得逆向工程或繞過付費機制
- 違反時保留終止帳號權利

## § 6 AI 服務聲明
{跟 Privacy § 7 一致}

## § 7 服務水準
- 本服務以 LINE 平台為投遞通道，投遞依 LINE 平台狀態
- 失敗會自動 retry 3 次
- 非本服務可控原因造成的延遲或失敗，不負賠償責任

## § 8 責任限制
本服務以「現狀」提供。除消費者保護法強制規定外，不負任何明示或默示擔保責任。

## § 9 準據法與管轄
依中華民國法律，由 {台北} 地方法院管轄。
```

---

## 3. Refund Policy（PAYUNi **強制檢查**）

**路徑**：`/policy/refund` + `/en/policy/refund`

### 寫死的要件
- 可退情境（明列）
- 不可退情境（明列）
- 處理時程（數字）
- 聯絡方式

### 骨架
```
# 退款政策

## 可退情境
- 訂閱當日（首次扣款 24 小時內）：全額退款
- 系統故障導致無法使用超過 24 小時：按比例退款
- 重複扣款（系統錯誤）：全額退款

## 不可退情境
- 已使用案點 / 已生成 AI 內容
- 訂閱期內超過 24 小時後申請（請改用「取消訂閱」）
- 違反服務條款被終止帳號者

## 處理時程
- 受理後 7 個工作日內審核
- 核准後 14 個工作日內退回原付款方式

## 取消訂閱（不需退款）
請至 /billing → 點「管理訂閱」→ 取消。取消後當期仍可使用至 period_end，
之後不再續扣。

## 申請方式
寄信至 support@yourdomain.com，主旨「退款申請」，內附：
- 訂單 ID（在 /billing 可查）
- 退款原因
- 註冊 email
```

---

## 4. Payment Info（PAYUNi **強制檢查**）

**路徑**：`/policy/payment-info` + `/en/policy/payment-info`

### 寫死的要件
- 「由 PAYUNi 統一金流安全處理」
- 「本服務不儲存信用卡資料」
- 受支付方式
- 發票處理（**個人 PAYUNi 註冊者寫「不開立統一發票」**）

### 骨架
```
# 付款說明

## 金流處理
本服務付款由台灣 PAYUNi 統一金流安全處理，
本服務不儲存信用卡卡號、CVV 或任何敏感付款資料。

## 受支付方式
- 信用卡（VISA / Mastercard / JCB / American Express）
- {如開：超商代碼繳費、ATM 轉帳}

## 訂閱扣款
訂閱會於 period_start 自動扣款，扣款失敗會於 3、5、7 天重試。
連續失敗將暫停服務直到付款方式更新。

## 發票
{個人 PAYUNi 註冊者}
本服務目前為個人經營，依《統一發票使用辦法》第 4 條，月銷售額未達起徵點
（NT$40,000）之服務業，不開立統一發票。如需收據，請於 /billing 下載
PAYUNi 交易明細。

{公司 / 行號 PAYUNi 註冊者}
本服務透過 ezPay 電子發票自動開立。請於 /billing 頁面填寫統編 / 抬頭。
電子發票將於扣款成功後 24 小時內寄至註冊 email。

## 幣別
所有金額以新台幣 (TWD) 計算。
```

---

## 5. Contact（客服資訊）

**路徑**：`/policy/contact` + `/en/policy/contact`

### 寫死的要件
- email（**自有域名**，不要 gmail / hotmail）
- 服務時間
- 平日 / 假日不同窗口（如有）

### 骨架
```
# 聯絡我們

## 客服信箱
support@yourdomain.com

## 服務時間
週一至週五 09:00-18:00（國定假日除外）

## 一般查詢
- 帳號 / 登入問題：support@yourdomain.com
- 付款 / 退款：billing@yourdomain.com
- 隱私 / 資料權利：privacy@yourdomain.com

## 預期回覆時間
- 一般查詢：1-2 個工作日
- 付款 / 退款：3-5 個工作日
- 緊急（帳號被盜）：當日

## 公司資訊（PAYUNi 註冊）
{個人}
- 經營者：{真實姓名}
- 聯絡地址：{備案地址}

{公司}
- 公司名稱：{}
- 統一編號：{}
- 公司地址：{}
- 負責人：{}
```

---

## 6. Index Footer（首頁底部 policy 連結）

**HTML 骨架**：
```html
<footer class="bg-gray-50 border-t border-gray-200 py-8">
  <div class="max-w-7xl mx-auto px-4 grid grid-cols-2 md:grid-cols-4 gap-4 text-sm">
    <div>
      <h4 class="font-bold mb-2">服務</h4>
      <a href="/billing">付款方案</a>
      <a href="/login">登入</a>
    </div>
    <div>
      <h4 class="font-bold mb-2">政策</h4>
      <a href="/policy/privacy">隱私權政策</a>
      <a href="/policy/terms">服務條款</a>
      <a href="/policy/refund">退款政策</a>
      <a href="/policy/payment-info">付款說明</a>
    </div>
    <div>
      <h4 class="font-bold mb-2">客服</h4>
      <a href="/policy/contact">聯絡我們</a>
      <a href="mailto:support@yourdomain.com">support@yourdomain.com</a>
    </div>
    <div>
      <h4 class="font-bold mb-2">語言</h4>
      <a href="/">繁體中文</a>
      <a href="/en">English</a>
    </div>
  </div>
  <div class="text-center text-xs text-gray-400 mt-6">
    Powered by PAYUNi · © 2026 {你的品牌}
  </div>
</footer>
```

---

## 對審查員的隱藏訊息

PAYUNi 審查員會抓的「形式 OK 但內容矛盾」陷阱：

1. **Terms 寫可退訂，但 UI 點不到取消按鈕** → 接 Recur Customer Portal
2. **Privacy § 7 寫不使用 AI，但 Terms § 6 寫 AI 自動回覆** → 對齊
3. **網站賣的是 A，PAYUNi 註冊營業項目是 B** → 砍模組或重註冊
4. **Refund 寫退款，但實際 webhook 沒寫 refund ledger** → 加 `col_refund_ledger`
5. **Payment Info 寫「可開統一發票」，但你註冊個人身份** → 改成「不開立」

過審前自己跑一遍上面 5 條，確認沒矛盾。
