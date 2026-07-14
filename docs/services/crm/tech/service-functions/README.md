# وظائف الخدمة (Use-cases)

كل use-case `final class` بـ `__invoke()` تحت `app/Services/Crm/<Group>/`.

## CalendarEvents
`CreateCalendarEvent` · `UpdateCalendarEvent` (دمج عبر `array_key_exists` ثم
`fresh()`) · `DeleteCalendarEvent` (soft-delete) · `ExpandRecurrences`
(يفكّ `FREQ=DAILY|WEEKLY|MONTHLY|YEARLY;INTERVAL=N` إلى نُسخ داخل نافذة، حدّ 730،
يحترم `recurrence_until`) · `BuildIcsFeed` (`VCALENDAR`: `VEVENT` لكل حدث +
`VTODO` لكل مهمة غير مكتملة، حدّ 500 صفّاً).

## Contracts
`RenderTemplate` (يستبدل `{{var}}` من سياق العقد/الحساب/العميل + `today`) ·
`SignContract` (يكتب `signature_png` + بيانات الموقّع + `signed_at` ويقلب
`status='signed'`).

## DocumentFolders
`CreateFolder` · `RenameFolder` · `DeleteFolder` (تعاودي عبر `descendantIds()`:
يحذف المستندات المتداخلة ثم المجلّدات من الأسفل للأعلى).

## Documents
`UploadDocument` (قرص `public` تحت `crm/documents/Y/m`؛ رفعات العميل المحتمل
تُوجَّه لمجلّد `Leads root` ثابت؛ إزالة تكرار العنوان داخل المجلّد) ·
`DeleteDocument` (يمسح الملف من القرص ثم soft-delete).

## TimeEntries
`StartTimer` (يوقف أي مؤقّت مفتوح للمستخدم ثم يفتح صفّاً جديداً) · `StopTimer`
(يختم `ended_at` ويحسب `duration_seconds` من الطوابع الخام).

## Inbox
`IssueEmbedToken` (JWT هاند-رولد HS256 لـ okta-whatsapp: `aud=okta-whatsapp`،
`scope=platform.admin`، TTL 300s؛ السرّ من `BridgeSettings::embedSsoSecret()`؛
البريد من `identifiers()` نوع email).

راجع [نماذج البيانات](../models/README.md) و[الدليل](../../guide/README.md).
