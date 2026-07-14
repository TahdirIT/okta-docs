# النظام المالي (Finance)

نظام مالي ومحاسبي داخلي لأوكتا، **landlord-only** (على مستوى المنصّة لا الجهات)،
يعيش في `app/Services/Finance/*` ويتكامل ثنائياً مع منصّة **وافق** المحاسبية.
مقصور على حاملي صلاحيات `finance.*` (نطاق system؛ دور `finance-admin` يملكها
كاملةً).

> **مصدر التفصيل:** `okta-web/CLAUDE.md` (قسم «النظام المالي») هو المرجع
> المُحدَّث؛ هذا الملف يلخّصه كموطن للخدمة في okta-docs. الصلاحيات نطاق system،
> تتبع معيار [`<feature>.<resource>.<action>`](../../tech-standards/permissions-naming.md).

## القرارات المعمارية

- **المكان**: `app/Services/Finance/<Resource>/<UseCase>.php` داخل `app/` الرئيسي (ليس Module).
- **العملة**: SAR افتراضياً (`decimal(15,2)` + `currency`).
- **الضريبة ودليل الحسابات**: يُداران في وافق؛ مرآة قراءة محلية `finance_wafeq_accounts`.
- **وافق**: يُعاد استخدام `app/Services/Wafeq/WafeqClient.php`. المزامنة عبر Queued Jobs
  (`tries=3`, backoff `[10,30,60]`) مع حقول حالة `wafeq_id/synced_at/sync_status/sync_error`.
- **Layout/Routes**: `layouts.finance` + `routes/finance.php` ببادئة `/finance/`.
- كل الجداول ببادئة `finance_*`؛ soft deletes + `created_by/updated_by` على الحسّاسة.

## المراحل (كلها مكتملة)

| المرحلة | تغطّي | خدمات رئيسية |
|---|---|---|
| **1 — الموردون + مرآة حسابات وافق** | مورد لكل (code تلقائي `V-YYYY-NNNN`)، مرآة دليل حسابات وافق للقراءة | `Vendors/{Create,Update,Delete,GenerateVendorCode}`، `Wafeq/{PushVendorToWafeq,SyncWafeqAccounts}` |
| **2 — حوافظ الصرف + محرّك اعتماد** | `PV-YYYY-NNNN`، آلة حالة `draft→pending_approval→approved→paid→reconciled` (+rejected/cancelled)، مسارات اعتماد قابلة للإعداد (per amount band، role XOR user)، مرفقات، سند القيد في وافق | `PaymentVouchers/*` (17 use-case)، `ApprovalWorkflow/*`، `Wafeq/PushVoucherToWafeq` |
| **3 — فواتير الموردين + المصروفات المتكرّرة** | `VI-YYYY-NNNN` + رقم فاتورة المورّد، ربط M2M بالحوافظ (سداد جزئي/متعدد)؛ قوالب متكرّرة تولّد فواتير + scheduler | `VendorInvoices/*`، `RecurringExpenses/*`، `ProcessRecurringExpensesJob` (`dailyAt('03:30')`) |
| **4 — العُهد (Custody)** | حساب عهدة دائم لكل (موظف، عملة) + دورات صرف (`CC-YYYY-NNNN`)، بنود مصروف، تسوية (return/rollover)، تقرير حركة | `Custody/*` (12 use-case)، `Wafeq/PushCustodySettlementToWafeq` |
| **5 — تقارير + لوحة + تصدير + إعدادات** | كشف حساب مورد، مصروفات حسب التصنيف، أعمار الاعتمادات، أرصدة العهد؛ لوحة `/finance/dashboard`؛ تصدير Excel (Maatwebsite) + PDF (dompdf) | `Reports/*`، `Exports/Finance/*`، `Livewire/Finance/{Dashboard,Reports/*}` |
| **6 — إعادة بناء الحوافظ (ملاحظات المحاسبة)** | مورد لكل بند، ضريبة inclusive/exclusive (`CalculateLineTotals`)، جدول `finance_payment_methods` مُدار، توقيع المستلم، مرفقات polymorphic، طباعة PDF | `PaymentVouchers/{CalculateLineTotals,StoreRecipientSignature}`، `PaymentMethods/*`، `Vendors/CreateVendorInline` |

## المسارات (`routes/finance.php` → `/finance/*`)

الموردون، حسابات وافق، حوافظ الصرف (+ `{hashid}` و`/print`)، مسارات الاعتماد،
فواتير الموردين، المصروفات المتكرّرة، حسابات ودورات العُهد (+ تقرير الحركة)،
وسائل الدفع، اللوحة، والتقارير الأربعة. ~30 صفحة Livewire تحت `app/Livewire/Finance/*`.

## الصلاحيات (نطاق system)

`finance.vendors.*`، `finance.wafeq_accounts.view`، `finance.wafeq.sync`،
`finance.payment_vouchers.{view,create,update,delete,submit,approve,reject,mark_paid,reconcile,cancel,print}`،
`finance.approval_workflows.*`، `finance.vendor_invoices.*`،
`finance.recurring_expenses.*`، `finance.custody_accounts.*`، `finance.custody.*`،
`finance.payment_methods.*`، `finance.tax_settings.*`، `finance.dashboard.view`،
`finance.reports.*`.

## إعدادات PlatformSetting المطلوبة

- `finance.wafeq.enabled` (افتراضي '1') · `finance.wafeq.api_key`
- `finance.payment_vouchers.cash_account_id` (حساب الخزينة في وافق — لازم لكل push)
- `finance.notifications.user_id` (مستلم احتياطي للإشعارات)
- `finance.tax.default_rate` (افتراضي '15') · `finance.tax.allow_inclusive_input` (افتراضي '1')

## ملاحظة تمييز

هذا **غير** [بوابة الدفع](../payment-gateway/README.md) (فوترة الجهات عبر Neoleap)
ولا مشغّل الدفع الموحّد للتطبيقات الشريكة — النظام المالي محاسبة داخلية على مستوى
المنصّة تتكامل مع وافق. `Wafeq/*` هنا (دفع الموردين) يختلف عن
`WafeqPaymentSyncService` (فوترة الاشتراكات) في بوابة الدفع.
