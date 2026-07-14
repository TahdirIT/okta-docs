# الاشتراكات ومميزات الباقات (Subscriptions & Plan Features)

نظامان متكاملان في `okta-web` يحكمان **ما تحصل عليه الجهة**: إدارة الاشتراكات
(الباقات، الفئات، دورة حياة الاشتراك، بوابات الدفع) ونظام **مميزات الباقات**
(toggle-per-plan للقدرات).

> يكمّلان نظام **modules** (المتجر) — راجع [`claude/web.md`](../../../claude/web.md).
> الفرق: **Permission** (مستخدم×دور) · **Module** (تطبيق مشحون للجهة) · **Plan
> Feature** (قدرة تتيحها الباقة). كثير من العمليات يجمع الثلاثة.

## إدارة الاشتراكات (`app/Services/SubscriptionManagement/*`)

- `Plans/` · `Tiers/` — تعريف الباقات وفئاتها.
- `Subscriptions/` + `Lifecycle/` — إنشاء/تجديد/تعليق/تفعيل/منح تجربة/فترة سماح
  (`trial|active|grace_period|suspended`)، وتثبيت موديولات الباقة تلقائياً.
- `Payments/` — بوابات الاشتراك (Tabby/Tamara/Neoleap/Manual) — منفصلة عن
  [بوابة الدفع](../payment-gateway/README.md) الأساسية.
- الواجهات: `Livewire/Settings/Subscriptions/*`، `Livewire/Tenant/Subscriptions/*`؛
  المسارات `/subscription` و`/finance/subscription-plans`.

## مميزات الباقات (Plan Features)

toggle لكل قدرة يمنحها superadmin للجهات حسب باقتها — يكمّل المتجر.

- **الجدول**: `subscription_plan_features` (landlord): `plan_id` + `feature_key` (unique) + `is_enabled`.
- **الكاتالوج**: `App\Services\Subscriptions\PlanFeatureCatalog::all()` — ثابت في الكود؛ لكل ميزة `group, label_ar/en, description, default_enabled, requires?`. أي ميزة جديدة تُضاف هنا فقط (لا migration).
- **Default**: غياب صفّ صريح ⇒ `default_enabled` من الكاتالوج (توافق عكسي).
- **البوابة**: `App\Services\Subscriptions\PlanGate::tenantHas(?Tenant, string $key): bool` — يحلّ الاشتراك النشط، يتأكد من حالته، ويراجع الصفّ + الكاتالوج. اختصار Eloquent: `$tenant->hasPlanFeature('landing.edit')`.

### نقاط الفحص الثلاث لأي ميزة

1. **السايدبار**: `'plan_feature' => '<key>'` في `config/sidebar.php`.
2. **`mount()`**: `abort_unless($tenant->hasPlanFeature('<key>'), 403, …)` بعد فحص الصلاحية.
3. **داخل الـ actions الحسّاسة**: نفس `abort_unless` قبل أي كتابة.

### الواجهة

صفحة `/finance/subscription-plans` بها زر **Features** → modal
`settings.subscriptions.plan-features-modal` (`PlanFeaturesModal`): switches
مجموعة حسب `group` مع تعطيل الـ child حين يكون `requires` معطّلاً.

### أمثلة جاهزة

`landing.edit`, `landing.manage_domains`, `landing.connect_custom_domain`
(requires manage_domains)، `hierarchy.console`, `hierarchy.reports`,
`hierarchy.bulk_install` (تتطلّب console). كلها `default ON`.

## التغطية المركزية (covers_children)

عمود `covers_children` على `subscriptions` (للجهات الحاوية): `PlanGate::tenantHas`
يصعد سلسلة `parent_tenant_id` عبر `Tenant::coveringSubscription()`، فتحترم بوابات
مميزات الباقات تغطية الجهة الأم تلقائياً. راجع
[كونسول الجهات الحاوية](../tenant-hierarchy/README.md).
