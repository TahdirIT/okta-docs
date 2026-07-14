# نماذج بيانات النظام المالي (Models)

كل الجداول ببادئة `finance_*` (landlord)، مبالغها `decimal(15,2)` + `currency`،
مع soft deletes و`created_by`/`updated_by` على الحسّاسة. مجمّعة حسب المرحلة.

## الموردون + مرآة وافق

| الجدول | أعمدة رئيسية |
|---|---|
| `finance_vendors` | `code` (`V-YYYY-NNNN`)، `type` (supplier/employee/referral_partner/government/other)، حقول بنكية وضريبية، `user_id` (موظف اختياري)، `created_inline`، حقول Wafeq (`wafeq_id`, `wafeq_synced_at`, `wafeq_sync_status`, `wafeq_sync_error`) |
| `finance_wafeq_accounts` | مرآة دليل حسابات وافق (قراءة محلية فقط) |

## حوافظ الصرف + محرّك الاعتماد

| الجدول | أعمدة رئيسية |
|---|---|
| `finance_payment_vouchers` | `code` (`PV-YYYY-NNNN`)، `vendor_id` (nullable)، `voucher_date`، `payment_method_id`، totals محسوبة، حقول الدفع (`paid_amount`, `bank_transaction_ref`, `recipient_name`, `paid_at`, `paid_by_user_id`)، `accountant_user_id`، `approved_at`، توقيع المستلم (`recipient_signature_disk/path`)، حقول Wafeq (`wafeq_journal_entry_id`, …) |
| `finance_payment_voucher_items` | `wafeq_account_id` (حساب المصروف)، `vendor_id`، `invoice_number/date`، `supplier_tax_number`، `amount`/`tax_amount`/`total_amount`، `tax_rate`، `is_tax_inclusive`، `order` |
| `finance_approval_workflows` | تعريف المسار لكل `document_type` (مسار نشط واحد) |
| `finance_approval_workflow_steps` | `min_amount`/`max_amount` (nullable=دائماً)، `approver_role_id` XOR `approver_user_id` |
| `finance_voucher_approvals` | لقطة الخطوات المنطبقة عند الإرسال + حالتها + سجل الإجراء (audit) |

## فواتير الموردين + المصروفات المتكرّرة

| الجدول | أعمدة رئيسية |
|---|---|
| `finance_vendor_invoices` | `invoice_number` (`VI-YYYY-NNNN`)، `supplier_invoice_number` (فريد لكل مورّد عبر Service)، `paid_amount`، `status` |
| `finance_voucher_invoice_link` | pivot M2M مع `applied_amount` (سداد على دفعات / حافظة تسدّد عدة فواتير) |
| `finance_recurring_expenses` | `vendor_id`، `frequency` (monthly/quarterly/yearly)، `start/end`، `next_run_at`، `lead_time_days`، `last_generated_at` |
| `finance_recurring_expense_items` | قوالب البنود (`default_amount`/`default_tax_amount` + `wafeq_account_id`) |

## العُهد (Custody)

| الجدول | أعمدة رئيسية |
|---|---|
| `finance_custody_accounts` | حساب دائم لكل (`user_id`, `currency`) unique، `wafeq_account_id`، `current_balance` |
| `finance_custody_cycles` | `code` (`CC-YYYY-NNNN`)، `issued_amount`، `issue_voucher_id` (FK لحافظة draft)، `purpose`، `status`، حقول التسوية (`settled_at`, `settled_remainder_amount`, `remainder_action: return\|rollover`) |
| `finance_custody_expenses` | بنود المصروف داخل دورة (+ `wafeq_account_id`) |
| `finance_custody_expense_attachments` | مرفقات لكل expense على disk `finance` |

## مشترك (المرحلة 6)

| الجدول | أعمدة رئيسية |
|---|---|
| `finance_payment_methods` | وسائل دفع مُدارة (`code` unique، `name_ar/en`، `icon`، `is_active`، `sort_order`) |
| `finance_attachments` | مرفقات polymorphic (`attachable_type/id`، `disk`, `path`, `original_name`, `mime_type`, `size_bytes`, `caption`) |

## Enums (آلات الحالة)

- `PaymentVoucherStatus`: `draft → pending_approval → approved → paid → reconciled` (+ `rejected`, `cancelled`).
- `CustodyCycleStatus`: `open → in_use → settled` أو `open|in_use → cancelled`.
- `RecurringFrequency`: `monthly | quarterly | yearly` (مع `advance()`).

راجع [آلات الحالة ودورات الحياة](../../guide/README.md).
