# الطبقة التقنية — النظام المالي

- [نماذج البيانات](./models/README.md) — جداول `finance_*` مجمّعة حسب المرحلة + الـ enums.
- [وظائف الخدمة (Use-cases)](./service-functions/README.md) — كل use-case لكل مورد + مزامنة وافق.

**Jobs** (queued، `tries=3`، backoff `[10,30,60]`): `PushVendorToWafeqJob`،
`PushVoucherToWafeqJob`، `PushCustodySettlementToWafeqJob`،
`ProcessRecurringExpensesJob` (`dailyAt('03:30')` مع `withoutOverlapping`).

**Storage**: disk `finance` (`storage/app/finance` أو S3 عبر `FINANCE_DISK_DRIVER`).

**Exports**: `app/Exports/Finance/*` (Excel عبر Maatwebsite) + قوالب PDF (dompdf)
في `resources/views/exports/finance/` تمتد من `_pdf-layout` (RTL، DejaVu Sans).
