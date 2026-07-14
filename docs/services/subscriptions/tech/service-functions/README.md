# وظائف الخدمة (Use-cases)

كل use-case `final class` بـ `__invoke()`. مجموعتان:
`app/Services/SubscriptionManagement/*` (الإدارة) و`app/Services/Subscriptions/*`
(بوابة المميزات).

## Plans
`CreatePlan` · `UpdatePlan` · `GetAvailablePlans(countryId, entityType)`
(الباقات النشطة لدولة ونوع كيان).

## Tiers
`CreateTier` · `UpdateTier` (فئات عدد الطلاب).

## Subscriptions
`CreateSubscription` · `GrantTrial` · `ActivateSubscription` · `ActivateFreeSubscription`
(باقة صفريّة بلا دفع) · `RenewSubscription` · `SuspendSubscription` ·
`AutoInstallPlanModules` (تثبيت موديولات الباقة) · `SyncPlanModuleExpiries`
(مواءمة انتهاء `tenant_module_installations` مع الاشتراك).

## Lifecycle (مجدولة)
`CheckExpiringSubscriptions(daysAhead=7)` · `ProcessGracePeriodExpiry`
(`grace_period → suspended` عند تجاوز `grace_ends_at`؛ عبر
`subscriptions:process-lifecycle`).

## Modules
`CreateSubscriptionModule` (إضافة مدفوعة) · `GetAvailableModules(entityType, countryId?)`.

## Payments
`ProcessPayment(invoice, method, data)` — يحلّ البوابة بـ `match($method)` ثم
`charge()`. البوابات `Gateways/{NeoleapGateway, TabbyGateway, TamaraGateway,
ManualGateway}` تنفّذ عقد `Contracts/PaymentGateway`.

## مميزات الباقات (`app/Services/Subscriptions/`)
`PlanGate::tenantHas(?Tenant, key)` — يحلّ الاشتراك (مع تغطية `covers_children`)
ويراجع الصفّ + `default_enabled`. `PlanFeatureCatalog::{all,has,defaultEnabled,
grouped}` — الكتالوج الثابت.

راجع [نماذج البيانات](../models/README.md) و[الدليل](../../guide/README.md).
