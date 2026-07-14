# الطبقة التقنية — الاشتراكات ومميزات الباقات

- [نماذج البيانات](./models/README.md) — جداول الاشتراك والباقات والفئات والفوترة
  ومميزات الباقات.
- [وظائف الخدمة (Use-cases)](./service-functions/README.md) — الباقات، الفئات،
  الاشتراكات، دورة الحياة، الموديولات، الدفع.

## البنية

- **إدارة الاشتراكات**: `app/Services/SubscriptionManagement/*` (كلها use-cases
  مستقلّة بـ `__invoke()`) مقسّمة إلى `Plans/`, `Tiers/`, `Subscriptions/`,
  `Lifecycle/`, `Modules/`, `Payments/`.
- **مميزات الباقات**: `app/Services/Subscriptions/{PlanGate,PlanFeatureCatalog}`.
- النماذج جميعها landlord-scoped؛ المبالغ `decimal(10,2)`، العملة الافتراضية `SAR`.

## بوابات دفع الاشتراك

- العقد `Payments/Contracts/PaymentGateway`:
  `charge(SubscriptionInvoice, data): SubscriptionPayment` +
  `refund(SubscriptionPayment, ?reason): bool`.
- البوابات `Payments/Gateways/{NeoleapGateway, TabbyGateway, TamaraGateway,
  ManualGateway}` تنفّذ العقد؛ التكامل الفعلي مع مزوّدات BNPL/البطاقات مُعلَّم
  كنقاط توسعة، و`ManualGateway` هو مسار الحوالة البنكية.
- الاختيار في `Payments/ProcessPayment` عبر `match($method)` على
  `neoleap|tabby|tamara|manual` (غيرها يرمي استثناءً).
- **منفصلة** عن [بوابة الدفع الأساسية](../../payment-gateway/README.md) في
  `app/Services/Payment/` (سطح المتجر — Neoleap).

## المسارات والواجهات

- **الجهة** (`routes/web.php`, باسم `tenant.subscription.*`): `/subscription`
  (`OverviewPage`) · `/subscription/plans` (`PlanSelectorPage`) ·
  `/subscription/payment` و`/subscription/renew` (`PaymentFlowPage`) · حالة الدفع
  (`routes/payment.php` → `PaymentStatusPage`).
- **إدارة المنصّة** (`routes/settings.php`، بادئة `/settings/`):
  `/settings/subscription-plans` (`Settings\Subscriptions\PlansPage` — يستضيف زر
  **Features** → `PlanFeaturesModal`) · `subscription-tiers` · `subscription-modules`
  · `subscriptions` (`SubscriptionsListPage`).
- Livewire: `Settings/Subscriptions/*` (`PlansPage`, `TiersPage`, `ModulesPage`,
  `SubscriptionsListPage`, `PlanModal`, `PlanPricingModal`, `PlanModulesModal`,
  `PlanFeaturesModal`, `TierModal`, `ModuleModal`, `SubscriptionModal` بها مفتاح
  `covers_children`) و`Tenant/Subscriptions/*` (`OverviewPage`, `PlanSelectorPage`,
  `PaymentFlowPage`, `PaymentModal`, `PaymentStatusPage`).

## الصلاحيات

- **مبذورة**: دورة اشتراكات المتجر `app_store.subscriptions.{view,create,cancel}`
  (نطاق منفصل).
- **حُرّاس مسارات**: صفحات إدارة الباقات محميّة بـ `permission:subscription-plans.manage`،
  وقائمة الاشتراكات + السياسة `SubscriptionPolicy` بـ `subscriptions.view`
  (إضافةً لـ `subscriptions.{create,update}` مُشار إليها في المكوّنات). تُستخدم
  كحُرّاس؛ راجع تزويدها في بيئتك قبل الاعتماد عليها.
