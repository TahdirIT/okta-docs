# وظائف الخدمة (Use-cases)

كل use-case ملف مستقل بـ `__invoke()` تحت `app/Services/Finance/<Resource>/`.
مجمّعة حسب المورد.

## Vendors
`CreateVendor` · `UpdateVendor` (يعيد المزامنة مع وافق) · `DeleteVendor` ·
`GenerateVendorCode` · `CreateVendorInline` (إضافة سريعة من نموذج الحافظة).

## PaymentVouchers
`GeneratePaymentVoucherCode` · `RecalculateVoucherTotals` · `CalculateLineTotals`
(ضريبة inclusive/exclusive) · `CreatePaymentVoucher` · `UpdatePaymentVoucher`
(draft فقط) · `SubmitForApproval` · `ApproveVoucher` (يعبّئ `accountant_user_id`
+ `approved_at` عند آخر خطوة) · `RejectVoucher` · `MarkAsPaid` (يدفع
`PushVoucherToWafeqJob`) · `ReconcileVoucher` · `CancelVoucher` ·
`DeletePaymentVoucher` (draft فقط) · `AttachFile` · `RemoveAttachment` ·
`StoreRecipientSignature` (ملف أو data-URL).

## ApprovalWorkflow
`CreateApprovalWorkflow` (يعطّل أي مسار نشط آخر لنفس `document_type`) ·
`UpdateApprovalWorkflow` · `DeleteApprovalWorkflow` · `ResolveApplicableSteps`
(الخطوات المنطبقة على المبلغ) · `GetPendingApprovalsForUser`.

## VendorInvoices
`GenerateVendorInvoiceCode` · `CreateVendorInvoice` · `UpdateVendorInvoice` ·
`CancelVendorInvoice` · `DeleteVendorInvoice` · `RecalculateVendorInvoicePaidAmount`
(يحدّث `paid_amount`/`status`) · `LinkVoucherToInvoice` (نفس المورد +
`applied_amount>0` + ضمن الرصيد والسعة) · `UnlinkVoucherFromInvoice`.

## RecurringExpenses
`CreateRecurringExpense` · `UpdateRecurringExpense` · `DeleteRecurringExpense` ·
`ToggleRecurringExpenseActive` · `GetDueRecurringExpenses` ·
`GenerateUpcomingInvoiceFromRecurring` (template → Vendor Invoice + يقدّم `next_run_at`).

## Custody
`GenerateCustodyCycleCode` · `CreateCustodyAccount` · `UpdateCustodyAccount` ·
`IssueCustodyCycle` (ينشئ الدورة + draft voucher عبر `CreatePaymentVoucher`) ·
`AddCustodyExpense` (active + amount ≤ remaining) · `RemoveCustodyExpense` ·
`AttachCustodyExpenseFile` · `RemoveCustodyExpenseAttachment` ·
`SettleCustodyCycle` (يستلزم `remainder_action` عند remainder>0 + يدفع
`PushCustodySettlementToWafeqJob`) · `CancelCustodyCycle` ·
`RecalculateCustodyBalance` · `GetCustodyMovementReport`.

## Reports
`VendorAccountStatement` · `ExpensesByCategory` · `PendingApprovalsAging` ·
`OpenCustodyBalances` · `FinanceDashboardSummary`.

## PaymentMethods
`CreatePaymentMethod` · `UpdatePaymentMethod` · `ActivatePaymentMethod` ·
`DeletePaymentMethod`.

## Wafeq (المزامنة)
`PushVendorToWafeq` · `SyncWafeqAccounts` · `PushVoucherToWafeq` (Journal Entry:
مدين حساب المصروف لكل بند، دائن حساب الخزينة `finance.payment_vouchers.cash_account_id`)
· `PushCustodySettlementToWafeq`. تُدفع عبر Queued Jobs
(`tries=3`, backoff `[10,30,60]`)، وتحترم `PlatformSetting('finance.wafeq.enabled')`
و`WafeqClient::isConfigured()`.
