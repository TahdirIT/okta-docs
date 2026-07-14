# نماذج بيانات الاشتراكات (Models)

النماذج في `app/Models/*` (landlord). المبالغ `decimal(10,2)`، العملة الافتراضية
`SAR`. أغلب الجداول soft-deletes.

## الاشتراك والباقات

| النموذج → الجدول | أعمدة رئيسية |
|---|---|
| `Subscription` → `subscriptions` | `tenant_id`، `plan_id`، `tier_id?`، `billing_period` (monthly/quarterly/semi_annual/annual)، **`status`** (trial/active/grace_period/suspended/cancelled، def trial)، **`covers_children`** (bool، def false)، `trial_ends_at`/`starts_at`/`ends_at`/`grace_ends_at`، `base_price`/`additional_seats`/`additional_seats_price`/`total_price`/`currency`، `granted_by?`، `suspended_at?`/`cancelled_at?` |
| `SubscriptionPlan` → `subscription_plans` | `supported_country_id`، `entity_type`، `name_ar/en`، `description_ar/en`، `is_active`/`is_featured`/`sort_order`، `trial_days`، `grace_period_days` (def 7)، `disabled_billing_periods` (json) |
| `SubscriptionStudentTier` → `subscription_student_tiers` | `name_ar/en`، `min_students?`/`max_students?`، `is_teacher_without_students`، `sort_order`، `is_active` |

`Subscription`: `ACCESSIBLE_STATUSES = [trial, active, grace_period]` + مساعدات
`isAccessible/isActive/isTrial/isGracePeriod/isSuspended` ونطاقات
`accessible()`/`expired()`.

## التسعير والموديولات

جداول مساندة: `subscription_plan_prices`، `subscription_additional_seat_prices`،
`subscription_modules`، `subscription_module_prices`، `subscription_module_items`،
`subscription_plan_modules`، `subscription_tenant_price_overrides`.

## الفوترة والدفع

| النموذج → الجدول | أعمدة رئيسية |
|---|---|
| `SubscriptionInvoice` → `subscription_invoices` | `subscription_id`، `invoice_number` (unique)، `subtotal`/`tax_amount`/`total_amount`/`currency`، `status` (pending/paid/failed/cancelled/refunded)، `due_date`/`paid_at?` (+ حقول Wafeq) |
| `SubscriptionPayment` → `subscription_payments` | `invoice_id`، `payment_method` (neoleap/tabby/tamara/manual)، `amount`/`currency`، `status` (pending/completed/failed/refunded)، `transaction_id?`، `payment_data` (json)، `processed_at?`/`processed_by?` |
| `SubscriptionPaymentReceipt` → `subscription_payment_receipts` | إثبات تحويل يدوي: `tenant_id`، `subscription_id?`/`subscription_payment_id?`، `bank_account_label/iban`، `transfer_amount`/`transfer_date`/`transfer_reference?`، `receipt_path`، `status` (pending/approved/rejected)، `reviewed_by?`/`reviewed_at?`/`rejection_reason?` |

## مميزات الباقات

| النموذج → الجدول | أعمدة رئيسية |
|---|---|
| `SubscriptionPlanFeature` → `subscription_plan_features` | `plan_id`، `feature_key`، `is_enabled` (def true)، unique(`plan_id`,`feature_key`) |

الكاتالوج نفسه ثابت في الكود (`PlanFeatureCatalog`) لا في جدول؛ الجدول يحمل
التجاوزات الصريحة فقط. راجع [الدليل — كتالوج المميزات](../../guide/README.md#كتالوج-المميزات-الحالي).
