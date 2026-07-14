# دليل الاشتراكات ومميزات الباقات

## دورة حياة الاشتراك

حالة `Subscription.status`:

```
trial ──▶ active ──▶ grace_period ──▶ suspended
             │                          
             └──────────────▶ cancelled 
```

- **الحالات المتاحة** (`ACCESSIBLE_STATUSES`): `trial`، `active`، `grace_period`
  — الجهة تصل خدماتها في هذه الثلاث فقط. `suspended`/`cancelled` تحجب.
- **منح تجربة**: `GrantTrial` ينشئ اشتراكاً `trial` بمدّة `plan.trial_days`.
- **التفعيل**: `ActivateSubscription` يقلبها `active` ويضبط التواريخ؛
  `ActivateFreeSubscription` يفعّل باقة صفريّة السعر بلا دفع.
- **التجديد**: `RenewSubscription` يمدّد `ends_at`.
- **فترة السماح ثم التعليق**: `ProcessGracePeriodExpiry` (مجدول عبر
  `subscriptions:process-lifecycle`) يقلب `grace_period → suspended` عند تجاوز
  `grace_ends_at`. `CheckExpiringSubscriptions(daysAhead)` يرصد المقتربة من
  الانتهاء.
- **موديولات الباقة**: `AutoInstallPlanModules` يثبّت موديولات المنصّة المرتبطة
  بالباقة تلقائياً، و`SyncPlanModuleExpiries` يوائم انتهاء
  `tenant_module_installations` مع الاشتراك.

## الباقات والفئات والأسعار

- `SubscriptionPlan` مرتبط بـ (`supported_country_id` + `entity_type`) وله
  `trial_days` و`grace_period_days` (افتراضي 7). `GetAvailablePlans(countryId,
  entityType)` يُرجع الباقات النشطة لدولة ونوع كيان.
- `SubscriptionStudentTier` = فئات عدد الطلاب (`min/max_students` أو
  `is_teacher_without_students`).
- جداول التسعير: `subscription_plan_prices`، `subscription_additional_seat_prices`،
  و`subscription_tenant_price_overrides` (تجاوزات لكل جهة).

## الفوترة والدفع

- `SubscriptionInvoice` (`subtotal`/`tax_amount`/`total`، حالة
  `pending|paid|failed|cancelled|refunded`) ← `SubscriptionPayment`
  (`payment_method` ∈ `neoleap|tabby|tamara|manual`، حالة
  `pending|completed|failed|refunded`).
- `ProcessPayment(invoice, method, data)` يحلّ البوابة بـ `match($method)` ثم
  ينادي `charge()`. البوابات الأربع تنفّذ عقد `PaymentGateway`؛ `charge()` ينشئ
  `SubscriptionPayment` بحالة `pending`، ونقطة الاتصال بمزوّدات
  Tabby/Tamara/Neoleap هي نقطة التوسعة، بينما `ManualGateway` هو مسار الحوالة
  البنكية.
- `SubscriptionPaymentReceipt` يحفظ إثباتات التحويل اليدوي (حالة
  `pending|approved|rejected` بمراجعة إداري).

## مميزات الباقات (Plan Features)

toggle لكل قدرة يمنحها superadmin للجهات حسب باقتها — يكمّل المتجر (Modules).

- **الجدول**: `subscription_plan_features` (landlord): `plan_id` + `feature_key`
  (unique) + `is_enabled`.
- **الكاتالوج**: `PlanFeatureCatalog::all()` ثابت في الكود؛ لكل ميزة
  `group, label_ar/en, description, default_enabled, requires?`. الميزة الجديدة
  تُضاف هنا فقط (لا migration).
- **الافتراضي**: غياب صفّ صريح ⇒ `default_enabled` من الكاتالوج.
- **البوابة** `PlanGate::tenantHas(?Tenant, key)`:
  1. لا جهة ⇒ `false`.
  2. اشتراك الجهة إن كانت حالته متاحة، وإلا **التغطية المركزية**
     `coveringSubscription()` (صعود `parent_tenant_id` لأوّل اشتراك متاح مفعَّل
     عليه `covers_children`).
  3. لا اشتراك ⇒ `false`؛ وإلا: صفّ صريح ⇒ `is_enabled`، وإلا `default_enabled`.
- اختصارات Eloquent: `$tenant->hasPlanFeature(key)`، `$plan->hasFeature(key)`.

### كتالوج المميزات الحالي

كل المفاتيح `default_enabled = true`:

| المفتاح | المجموعة | يتطلّب |
|---|---|---|
| `landing.edit` | landing | — |
| `landing.manage_domains` | landing | — |
| `landing.connect_custom_domain` | landing | `landing.manage_domains` |
| `notifications.whatsapp.enabled` | notifications | — |
| `messaging.broadcast` | notifications | — |
| `notifications.partner_providers` | notifications | — |
| `partner_notifications` | partner_apps | — |
| `payments.partner_providers` | partner_apps | — |
| `daycare.operations_app` | daycare | — |
| `daycare.daily_reports` | daycare | `daycare.operations_app` |
| `daycare.check_in_out` | daycare | `daycare.operations_app` |
| `hierarchy.console` | hierarchy | — |
| `hierarchy.reports` | hierarchy | `hierarchy.console` |
| `hierarchy.bulk_install` | hierarchy | `hierarchy.console` |

### نقاط الفحص الثلاث لأي ميزة

1. **السايدبار**: `'plan_feature' => '<key>'` في `config/sidebar.php`.
2. **`mount()`**: `abort_unless($tenant->hasPlanFeature('<key>'), 403, …)` بعد
   فحص الصلاحية.
3. **داخل الـ actions الحسّاسة**: نفس `abort_unless` قبل أي كتابة.

## التغطية المركزية (covers_children)

عمود `covers_children` (bool) على `subscriptions` للجهات الحاوية: `PlanGate`
يصعد سلسلة `parent_tenant_id` عبر `Tenant::coveringSubscription()`، فتحترم بوابات
مميزات الباقات تغطية الجهة الأم تلقائياً. راجع
[كونسول الجهات الحاوية](../../tenant-hierarchy/README.md).
