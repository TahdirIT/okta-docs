# `tenant_registration_sessions` (اقتراح غير مُنفَّذ)

> 🚧 **غير مُنفَّذ في okta-web.** لا يوجد جدول ولا نموذج بهذا الاسم. المعالج
> الحالي (`App\Livewire\Pages\Auth\RegisterTenant`) يحفظ حالة المسودّة في **جلسة
> Laravel** (مفتاح `register_tenant_wizard`) ولا يكتب لقاعدة البيانات إلا عند
> نجاح التسجيل. هذا المستند تصميم اختياري لو احتيج حفظ قابل للاستئناف عبر الجلسات
> — يُقرأ كاقتراح لا كواقع.

## الغرض

دعم تجربة Wizard متعددة الخطوات مع حفظ مؤقت/قابل للاستئناف (Draft) خصوصًا عندما:

- تكون الحقول ديناميكية حسب الدولة والنوع
- توجد خطوات تحقق/توثيق لاحقة
- نحتاج ربط “محاولة التسجيل” بالأحداث (Events) والتتبع (Analytics)

## أعمدة مقترحة

- **id**: `bigint` (PK)
- **ulid**: `char(26)` unique
- **country_id**: `bigint` nullable
- **tenant_type**: `varchar` nullable
- **education_level_group_id**: `bigint` nullable
- **draft_payload**: `jsonb` (مدخلات wizard بدون كلمة المرور/بيانات حساسة)
- **status**: `varchar` default `draft` (مثل: `draft | submitted | created | cancelled`)
- **owner_user_id**: `bigint` nullable (FK → `users.id`) — عند “لدي حساب” أو بعد إنشاء المستخدم
- **tenant_id**: `bigint` nullable (FK → `tenants.id`) — بعد الإنشاء
- **created_at / updated_at**: `timestamptz`

## ملاحظات

- يجب عدم تخزين كلمة المرور أو OTP أو أسرار حساسة داخل `draft_payload`.

## حالة التنفيذ في `okta-web`

- هذا الجدول/الموديل **غير موجود حاليًا** في `okta-web` (لا يوجد migration ولا Model مطابق).
- اعتبره **اقتراح تصميم** لدعم UX الـ wizard في `tenant-registration` إذا احتجنا “حفظ مسودة/استئناف” أو تتبع التحقق.
