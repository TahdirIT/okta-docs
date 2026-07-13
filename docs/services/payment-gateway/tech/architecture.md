---
title: هندسة خدمة بوابات الدفع
---

## تدفق البيانات الكامل

```
المستخدم
  │
  ▼
POST /payment/checkout
  │  (PaymentIntent DTO)
  ▼
PaymentProcessor.initiateCheckout()
  ├─ إنشاء Payment (status: pending)
  ├─ حساب عمولة البوابة (GatewayFeeCalculator)
  ├─ PaymentGatewayFactory.makeFromConfig()
  │     └─ NeoleapProvider
  └─ createCheckoutSession() → Neoleap API
         └─ إعادة redirectUrl للمستخدم

المستخدم يُكمل الدفع على صفحة Neoleap
  │
  ├─ Callback (مزامن): GET /payment/neoleap/callback
  │    └─ verifyPayment(sessionId)
  │         └─ processSuccessfulPayment() أو processFailedPayment()
  │
  └─ Webhook (غير مزامن): POST /webhooks/neoleap
       ├─ validateWebhookSignature()
       ├─ تسجيل PaymentWebhookLog
       └─ ProcessPaymentWebhook Job (queue)
            └─ processSuccessfulPayment() أو processFailedPayment()

processSuccessfulPayment()
  ├─ تحديث Payment (status: completed, paid_at)
  └─ إطلاق PaymentCompleted Event
       ├─ SyncPaymentToWafeqListener → SyncPaymentToWafeq Job
       │    └─ WafeqPaymentSyncService.syncPaymentToWafeq()
       │         ├─ resolveContact() — البحث أو إنشاء جهة الاتصال في وافق
       │         ├─ createInvoice() — فاتورة
       │         ├─ recordPayment() — سند قبض
       │         └─ createFeeJournalEntry() — قيد عمولة (إن fee > 0)
       ├─ ActivateSubscriptionListener
       │    └─ تفعيل الاشتراك / الوحدة حسب payable_type
       └─ SendPaymentReceiptListener
            └─ PaymentSuccessNotification → إيميل للعميل
```

---

## نموذج قاعدة البيانات

### جدول `payments`

```
id | tenant_id | uuid | payable_type | payable_id
   | payment_gateway_id | gateway_reference | gateway_session_id
   | amount | vat_amount | total_amount | currency
   | gateway_fee_percentage | gateway_fee_fixed | gateway_fee_total | net_amount
   | status | failure_reason
   | wafeq_invoice_id | wafeq_payment_id | wafeq_fee_entry_id | wafeq_synced_at
   | paid_at | created_at | updated_at
```

### جدول `tenant_payment_gateway_configs`

```
id | tenant_id | payment_gateway_id
   | credentials (encrypted JSON)
   | environment (sandbox/production)
   | is_enabled | is_tested
   | wafeq_collection_account_id | wafeq_revenue_account_id | wafeq_tax_rate_id
   | fee_percentage | fee_fixed_amount | fee_currency | wafeq_fee_expense_account_id
```

---

## مزودو الدفع (Providers)

كل بوابة دفع تُنفِّذ `PaymentProviderInterface`:

```php
interface PaymentProviderInterface
{
    public function createCheckoutSession(PaymentIntent $intent): CheckoutSession;
    public function verifyPayment(string $sessionId): PaymentResult;
    public function handleWebhook(Request $request): WebhookResult;
    public function refund(Payment $payment, ?float $amount = null): RefundResult;
    public function supportsRecurring(): bool;
    public function validateWebhookSignature(Request $request): bool;
}
```

يُحدَّد المزود عبر `provider_class` في جدول `payment_gateways`. يُنشئ `PaymentGatewayFactory` المزود المناسب حسب `slug` البوابة وإعدادات المستأجر.

---

## الـ DTOs (كائنات نقل البيانات)

| الكلاس | الغرض |
|--------|-------|
| `PaymentIntent` | وصف ما يريد المستخدم دفعه |
| `CheckoutSession` | نتيجة إنشاء جلسة الدفع (sessionId + redirectUrl) |
| `PaymentResult` | نتيجة التحقق من الدفع (success/failure + reference) |
| `WebhookResult` | نتيجة تحليل Webhook |
| `RefundResult` | نتيجة طلب استرداد |
| `GatewayFeeBreakdown` | تفصيل حساب العمولة |

---

## استراتيجية التجديد التلقائي

تعتمد على ما إذا كانت البوابة تدعم الـ Tokenization:

**بوابة تدعم Tokenization:**
- تخزين رمز البطاقة (ليس الرقم) في `tenant_payment_method_tokens`
- `ProcessSubscriptionRenewals` (مهمة يومية) تُجدد الاشتراكات تلقائياً
- عند فشل الشحن: إعادة المحاولة 3 أيام، ثم إشعار + تعليق

**بوابة لا تدعم Tokenization (مثل Neoleap حالياً):**
- إرسال تذكيرات إيميل: قبل 7 أيام، 3 أيام، يوم واحد
- إنشاء رابط دفع في الإيميل
- فترة سماح: 3 أيام بعد انتهاء الاشتراك
- بعد فترة السماح: تعليق الوصول

---

## الأمان

- **توقيع Webhook**: HMAC-SHA256 عبر `X-Neoleap-Signature` header
- **تشفير البيانات**: بيانات الاعتماد مشفرة بـ `Crypt::encryptString()`
- **CSRF**: مستبعد من مسارات الـ Webhook فقط
- **Idempotency**: `gateway_session_id` يمنع المعالجة المزدوجة
- **لا أرقام بطاقات**: يُخزَّن رمز الـ Token فقط عند دعم الـ Tokenization
