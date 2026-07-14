# دليل النظام المالي — الدورات وآلات الحالة

سرد لدورات الحياة الأساسية. للجداول راجع [tech/models](../tech/models/README.md)،
وللـ use-cases [tech/service-functions](../tech/service-functions/README.md).

## حافظة الصرف + محرّك الاعتماد

```
draft ─▶ pending_approval ─▶ approved ─▶ paid ─▶ reconciled
  │              │
  └─ cancelled   └─ rejected
```

- تُنشأ الحافظة `draft` ببنود، كل بند بحساب مصروف وافقي ومبلغ (ضريبة
  inclusive/exclusive عبر `CalculateLineTotals`).
- `SubmitForApproval` يبني **لقطة** من مسار الاعتماد النشط لنوع المستند: يحلّ
  `ResolveApplicableSteps` الخطوات المنطبقة على المبلغ (نطاقات `min/max`)، كل خطوة
  معتمِدها `role` أو `user`. اعتماد آخر خطوة ⇒ `approved` + تعبئة
  `accountant_user_id`/`approved_at`. رفض أي خطوة يُسكِّت البقية.
- `MarkAsPaid` ينقلها `paid` ويدفع سند القيد إلى وافق (مدين المصروف / دائن الخزينة).
- `reconciled` تسوية نهائية.

مسار الاعتماد يُعرَّف في `finance_approval_workflows`/`_steps` (مسار نشط واحد لكل
`document_type`)؛ اللقطة في `finance_voucher_approvals` هي سجل التدقيق.

## فاتورة المورّد + الربط بالحوافظ

`received → matched → partially_paid → fully_paid` (+ `cancelled`).
`finance_voucher_invoice_link` (M2M مع `applied_amount`) يدعم: سداد فاتورة على
دفعات، وحافظة واحدة تُسدِّد عدة فواتير. `LinkVoucherToInvoice` يفرض نفس المورد و
`applied_amount>0` وعدم تجاوز رصيد الفاتورة أو سعة الحافظة، و
`RecalculateVendorInvoicePaidAmount` يحدّث الحالة تلقائياً.

## المصروفات المتكرّرة

قالب (`finance_recurring_expenses` + بنود) يولّد فاتورة مورّد (`received`) عند
الاستحقاق ويقدّم `next_run_at` بفترة واحدة. `ProcessRecurringExpensesJob`
(`dailyAt('03:30')`) يستخرج المستحقات ويُشعر حاملي `finance.vendor_invoices.view`.

## دورة العُهدة (Custody)

حساب دائم لكل (موظف، عملة)، ودورات منفصلة لكل صرف:

```
open ─▶ in_use ─▶ settled
  └──────┴─▶ cancelled
```

- `IssueCustodyCycle` ينشئ الدورة + حافظة draft (vendor=الموظف، مبلغ=`issued_amount`).
- أول `AddCustodyExpense` ينقلها `in_use` (amount ≤ remaining).
- `SettleCustodyCycle` يستلزم `remainder_action` عند بقاء رصيد: `return` (يُعاد
  للخزينة، يصفّر) أو `rollover` (يُبقى remainder)، ويدفع سند تسوية وافقي.
- `RecalculateCustodyBalance` يعيد حساب `current_balance`؛ `GetCustodyMovementReport`
  صفّ زمني (issue/expense/return) برصيد جارٍ.
