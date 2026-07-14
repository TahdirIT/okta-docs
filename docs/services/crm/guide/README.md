# دليل CRM — الأنابيب والعقود والتقويم والمستندات

منظومة CRM سطح منصّة (platform-level): كل من يحمل أحد أدوار CRM يعمل خارج سياق
أي جهة (الـ middleware يُجبِر سياق الفريق على `null`). التالي شرح المسارات الوظيفية.

## أنبوب العملاء المحتملين (Leads Pipeline)

- `Lead` يقع في **مرحلة** (`crm_pipeline_stages`) قابلة للإدارة من
  `/crm/settings/pipeline`؛ لكل مرحلة `order` ولون وأعلام `is_won`/`is_lost`.
- الحقول الأساسية: `title`، الجهة/جهة الاتصال، `source`، `priority`
  (`low/medium/high/urgent`)، `expected_value` + `currency`،
  `expected_close_date`، والمُسنَد إليه.
- `last_activity_at` يُحدَّث مع كل نشاط لترتيب الأنبوب حسب الحيوية.
- **التحويل**: `LeadConvert` يختم `converted_at`/`converted_by` ويربط العميل
  المحتمل بحساب/جهة اتصال.
- **الأنشطة** (`crm_activities`): مكالمة/بريد/اجتماع/ملاحظة/تغيير مرحلة/مهمة/واتساب،
  مع ربط جهات الاتصال عبر pivot `crm_activity_contact`.

## العقود والتوقيع الإلكتروني

دورة حالة العقد (`crm_contracts.status`):

```
draft ─▶ sent ─▶ signed ─▶ active ─▶ expired
   └────────────────────────────────▶ cancelled
```

1. **القوالب**: `crm_contract_templates` تحمل `body_html` بمتغيّرات `{{var}}` +
   قائمة `variables` (json). `RenderTemplate` يستبدل المتغيّرات من سياق
   العقد/الحساب/العميل المحتمل + `today`/`today_human`.
2. **التوقيع**: `SignContract` يكتب `signature_png` (لوحة توقيع)، `signed_by_name`،
   `signed_ip`، `signed_user_agent`، `signed_at=now()`، ويقلب `status='signed'`.
3. **الطباعة**: `ContractPdfController` و`ContractTemplatePdfController` يولّدان PDF.

## التقويم — التكرار وتغذية ICS

- `crm_calendar_events`: `starts_at`/`ends_at`/`all_day`، نوع
  (`meeting/call/visit/reminder/deadline/other`)، لون، `reminder_at`، وربط
  اختياري بعميل محتمل/حساب/مُسنَد إليه.
- **التكرار**: `rrule` (varchar 64) + `recurrence_until`. `ExpandRecurrences`
  يفكّ مجموعة فرعية من RRULE (`FREQ=DAILY|WEEKLY|MONTHLY|YEARLY;INTERVAL=N`) إلى
  نُسخ داخل نافذة زمنية (حدّ صارم 730 نسخة، يحترم `recurrence_until`).
- **تغذية ICS**: `BuildIcsFeed` يبني `VCALENDAR` كاملاً — `VEVENT` لكل حدث (مع
  `RRULE` عند التكرار) + `VTODO` لكل مهمة غير مكتملة. يُخدَم عبر رابط عام
  محميّ برمز: `/crm/calendar/feed/{user}/{token}.ics`.

## المستندات والمجلّدات

- `crm_documents` polymorphic (`documentable`)، تُخزَّن على قرص `public` تحت
  `crm/documents/Y/m`. الرفع يُزيل تكرار العنوان داخل المجلّد، ورفعات العميل
  المحتمل تُوجَّه آلياً إلى مجلّد ثابت `Leads root / #id — title`.
- `crm_document_folders` شجرة ذاتية المرجعية (`parent_id`). `DeleteFolder`
  تعاودي: يحذف المستندات المتداخلة (مع تنظيف القرص) ثم المجلّدات من الأسفل للأعلى.

## تتبّع الوقت

`crm_time_entries`: `StartTimer` يوقف أي مؤقّت مفتوح للمستخدم ثم يفتح صفّاً جديداً
(`started_at=now()`, task/lead/account, billable)؛ `StopTimer` يختم `ended_at`
ويحسب `duration_seconds` من الطوابع الخام.

## صندوق واتساب (Inbox)

مضمَّن عبر iframe من okta-whatsapp — راجع [الطبقة التقنية](../tech/README.md)
و[صفحة الخدمة](../README.md).
