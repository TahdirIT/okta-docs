# وظائف/Endpoints - Tenant Registration

هذا المستند يقترح وظائف/Endpoints منطقية لخدمة تسجيل الكيان (قد تختلف حسب البنية الفعلية للـ API).

## 1) جلب إعدادات التسجيل حسب الدولة

**الهدف**: تمكين الواجهة من بناء الـ Wizard ديناميكيًا.

- **Function**: `getEntityRegistrationCustomization(country_id)`
- **Returns**:
  - `contact_requirements`: `{ username_required, national_id_required, mobile_required, email_required }`
  - `enabled_tenant_types`: قائمة بأنواع الكيانات المفعلة (مرتبة)
  - `custom_fields_schema`: تعريف الحقول المخصصة + شروط الظهور
  - `stages` (اختياري): قائمة/شجرة مراحل الدولة لاستخدامها عند اختيار نوع كيان يتطلب مرحلة

> ملاحظة: المراحل مبنية على مفهوم “إدارة المراحل” في:
> `docs/services/countries-management/guide/stages-management.md`

## 2) تسجيل كيان جديد

- **Function**: `registerTenant(payload)`
- **Payload**:
  - `country_id`
  - `tenant_type`
  - `stage_id` (مطلوب فقط لأنواع: مدرسة إدارية، معلم فردي)
  - `tenant_name`
  - `custom_fields`
  - `owner`: `{ username, name, national_id?, mobile?, email?, password, password_confirmation }`
- **Behavior**:
  - validation شامل حسب إعدادات الدولة + schema
  - validation للمرحلة عند الأنواع التي تتطلبها (ضمن مراحل الدولة)
  - إنشاء Tenant + Owner user
  - وضع حالة `pending_verification`
  - إطلاق events (انظر `guide/05-events.md`)

## 3) بدء/إعادة إرسال تحقق الجوال

- **Function**: `startMobileVerification(user_id | session)`
- **Behavior**:
  - rate limiting
  - إطلاق `tenant_registration.verification.mobile_started.v1`

## 4) بدء/إعادة إرسال تحقق البريد

- **Function**: `startEmailVerification(user_id | session)`
- **Behavior**:
  - rate limiting
  - إطلاق `tenant_registration.verification.email_started.v1`

## 5) تأكيد التحقق

- **Function**: `confirmMobileVerification(code)` / `confirmEmailVerification(token)`
- **Behavior**:
  - تحديث flags `mobile_verified` / `email_verified`
  - إطلاق events `verified.*`
  - إذا تحققت قاعدة التفعيل: إطلاق `tenant_registration.activated.v1`

