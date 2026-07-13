---
title: بوابات الدفع والمحاسبة
---

## نظرة عامة على الخدمة

تتيح خدمة **بوابات الدفع** للمنصة استقبال المدفوعات من المستأجرين (المدارس/الجهات) عبر بوابات دفع متعددة، مع ربط كل عملية دفع تلقائياً بنظام **وافق** المحاسبي.

### ما الذي تقدمه الخدمة؟

- **دفع عبر صفحة Neoleap** — توجيه المستخدم إلى صفحة دفع Neoleap ARB المستضافة، وتلقي نتيجة الدفع عبر Callback وWebhook.
- **معالجة المدفوعات** — تسجيل الدفعة في قاعدة البيانات، حساب عمولة البوابة، وتحديث حالة الاشتراك.
- **تكامل وافق** — إنشاء فاتورة، سند قبض، وقيد عمولة البوابة تلقائياً في وافق عقب كل دفعة ناجحة.
- **دعم أنواع الدفع الأربعة** — اشتراك المنصة، اشتراك تطبيق، دفعة لمرة واحدة، وتجديد تلقائي.

---

## مكونات الخدمة

### 1. جداول قاعدة البيانات

| الجدول | الوصف |
|--------|-------|
| `payment_gateways` | بوابات الدفع المتاحة في المنصة (Neoleap، Tamara، Tabby) |
| `tenant_payment_gateway_configs` | إعدادات البوابة لكل مستأجر (بيانات الاتصال، الربط المحاسبي، العمولة) |
| `payments` | سجل المدفوعات الرئيسي |
| `payment_webhook_logs` | سجل Webhooks الواردة من البوابات |

### 2. بوابة Neoleap ARB

**نوع التكامل:** Hosted Payment Page (صفحة دفع مستضافة)

**تدفق الدفع:**
1. يطلب المستخدم الدفع → `POST /payment/checkout`
2. المنصة تُنشئ جلسة في Neoleap → تُعيد رابط الدفع
3. المستخدم يُكمل الدفع على صفحة Neoleap
4. Neoleap يُعيد المستخدم إلى `GET /payment/neoleap/callback`
5. Neoleap يُرسل Webhook غير متزامن إلى `POST /webhooks/neoleap`
6. تتم معالجة أي منهما أولاً (idempotent)

### 3. خط معالجة وافق

عقب كل دفعة ناجحة، يُنشئ `WafeqPaymentSyncService` في وافق:

```
الدفعة الناجحة
    ├─ فاتورة (Invoice) — ربط إيراد المبيعات
    ├─ سند قبض (Payment Record) — ربط حساب التحصيل (بوابة الدفع neoleap)
    └─ قيد عمولة — مصروفات ← حساب التحصيل (عمولة البوابة)
```

---

## هيكل الملفات

```
app/
├── Contracts/
│   └── PaymentProviderInterface.php          # واجهة مزودي الدفع
├── Services/Payment/
│   ├── PaymentProcessor.php                  # المنسق الرئيسي
│   ├── PaymentGatewayFactory.php             # مصنع مزودي الدفع
│   ├── GatewayFeeCalculator.php              # حاسبة عمولة البوابة
│   ├── DTOs/                                 # كائنات نقل البيانات
│   ├── Providers/
│   │   └── NeoleapProvider.php               # مزود Neoleap ARB
│   └── Wafeq/
│       └── WafeqPaymentSyncService.php       # مزامنة المدفوعات مع وافق
├── Models/
│   ├── Payment.php
│   ├── PaymentGateway.php
│   ├── TenantPaymentGatewayConfig.php
│   └── PaymentWebhookLog.php
├── Http/Controllers/
│   ├── PaymentController.php                 # Checkout + Callback
│   └── PaymentWebhookController.php          # Webhooks
├── Jobs/
│   ├── SyncPaymentToWafeq.php
│   ├── ProcessPaymentWebhook.php
│   └── ProcessSubscriptionRenewals.php
├── Events/
│   ├── PaymentCompleted.php
│   ├── PaymentFailed.php
│   └── SubscriptionActivated.php
└── Listeners/
    ├── SyncPaymentToWafeqListener.php
    ├── ActivateSubscriptionListener.php
    └── SendPaymentReceiptListener.php
```

---

## الإعداد

### متطلبات البيئة

1. تشغيل الـ migrations: `php artisan migrate`
2. تشغيل الـ seeder: `php artisan db:seed --class=PaymentGatewaysSeeder`
3. إعداد بوابة Neoleap عبر واجهة الإعدادات: **الإعدادات → المالية → Neoleap Gateway**

### إعداد Neoleap في الإعدادات

| الحقل | الوصف |
|-------|-------|
| API Key | مفتاح الوصول لـ API |
| Merchant ID | معرّف التاجر |
| Webhook Secret | مفتاح التحقق من توقيع الـ Webhook |
| البيئة | sandbox أو production |
| معرّف حساب التحصيل | حساب "بوابة الدفع neoleap" في وافق |
| معرّف حساب الإيراد | حساب "المبيعات" في وافق |
| معرّف نسبة الضريبة | نسبة ضريبة القيمة المضافة في وافق |
| نسبة العمولة % | نسبة عمولة البوابة (مثال: 2.5) |
| مبلغ ثابت إضافي | مبلغ ثابت فوق النسبة (مثال: 1.00 ر.س) |
| حساب مصروفات العمولة | حساب مصروفات العمولة في وافق |

---

## صيغة حساب العمولة

```
عمولة_إجمالية = (المبلغ_الإجمالي × نسبة_العمولة) + مبلغ_ثابت
صافي_المبلغ   = المبلغ_الإجمالي - عمولة_إجمالية

مثال:
  المبلغ        = 500.00 ر.س
  نسبة العمولة = 2.5% → 0.0250
  مبلغ ثابت    = 1.00 ر.س
  العمولة       = (500 × 0.025) + 1.00 = 13.50 ر.س
  الصافي        = 500.00 - 13.50 = 486.50 ر.س
```

---

## ملاحظات التطوير

- **بيانات الدفع**: جميع حقول المبالغ من نوع `DECIMAL(12,2)` — لا تُستخدم `FLOAT` أبداً.
- **بيانات الاعتماد**: مشفرة باستخدام `Crypt::encryptString()` في عمود `credentials`.
- **أرقام البطاقات**: لا تُخزَّن أبداً — رموز Tokenization فقط إن كانت مدعومة.
- **Idempotency**: معالجة الدفعة مضمونة من التكرار عبر `gateway_session_id`.
- **Webhooks**: جميع الـ payloads تُخزَّن في `payment_webhook_logs` قبل المعالجة.

---

## المراجع

- [`app/Contracts/PaymentProviderInterface.php`](../../../okta-web/app/Contracts/PaymentProviderInterface.php)
- [`app/Services/Payment/PaymentProcessor.php`](../../../okta-web/app/Services/Payment/PaymentProcessor.php)
- [`app/Services/Payment/Providers/NeoleapProvider.php`](../../../okta-web/app/Services/Payment/Providers/NeoleapProvider.php)
- [`app/Services/Payment/Wafeq/WafeqPaymentSyncService.php`](../../../okta-web/app/Services/Payment/Wafeq/WafeqPaymentSyncService.php)
