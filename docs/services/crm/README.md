# نظام إدارة العلاقات (CRM)

منظومة CRM على مستوى المنصّة في `okta-web`: حسابات، عملاء محتملون، جهات اتصال،
عقود (مع توقيع إلكتروني)، تقويم، مستندات، مهام، سجلات وقت، فواتير، لوحة، وبحث —
إضافةً إلى **صندوق واتساب** مضمَّن. المصدر: `app/Services/Crm/*`،
`routes/crm.php`، `routes/inbox.php`.

> توثيق مبدئي مبنيّ على جرد الكود؛ صندوق الوارد موثَّق تفصيلاً في `okta-web/CLAUDE.md`.

## المكوّنات

- **الخدمات** (`app/Services/Crm/*`): `Contracts` (+ e-sign/templates)، `Documents`،
  `DocumentFolders`، `CalendarEvents` (recurrence/ICS)، `TimeEntries`، `Inbox`.
- **الواجهات** (`Livewire/Crm/*`): `Accounts`، `Leads`، `Contacts`، `Contracts`،
  `Calendar`، `Documents`، `Tasks`، `Invoices`، `Dashboard`، `Search`، `Settings`.
- **المسارات**: `routes/crm.php` → `/crm/*`؛ `routes/inbox.php` → `/crm/inbox`.

## صندوق واتساب (CRM Inbox) — مضمَّن عبر iframe

صندوق محادثات واتساب على مستوى المنصّة **يُضمَّن مباشرةً عبر iframe** من
okta-whatsapp بدلاً من بناء UI محلي (الواجهة هناك مكتملة ومُجرَّبة):

- **الصفحة**: `GET /crm/inbox` → `App\Livewire\Crm\Inbox\InboxFrame` — يُصدر JWT ويضع iframe ملء الارتفاع.
- **iframe src**: `<connect_base_url>/embed/sso?token=<jwt>&return=/admin/inbox?embedded=1`.
- **JWT**: `App\Services\Crm\Inbox\IssueEmbedToken` (HS256 هاند-رولد؛ `iss=okta-web, aud=okta-whatsapp, scope=platform.admin, exp=iat+300`).
- **السر المشترك**: `BridgeSettings::KEY_EMBED_SSO_SECRET` (`comms.connect.embed_sso_secret`) مُشفَّر في `platform_settings`، يُضبط من بطاقة **Comms (Connect)** في `/settings/platform-delivery`.
- **Unread badge**: `<livewire:crm.inbox.unread-badge />` — الـ iframe يُرسل `postMessage({type:"inbox.unread", count})` يُفلتَر بالـ origin ويُعاد بثّه كحدث Livewire.
- **لا مرآة محلية**: حُذفت جداول `inbox_*` وخدماتها؛ أي استعلام محادثات يتمّ من okta-whatsapp مباشرة.
- **الصلاحيات**: `crm.inbox.view` (الصفحة) و`crm.inbox.manage_settings` (إعدادات Connect) — نطاق system.

## للتوثيق العميق لاحقاً

بنية نماذج الحسابات/العملاء/العقود، دورة توقيع العقد الإلكتروني، تكرار أحداث
التقويم وتصدير ICS، ونموذج الفواتير — تُوثَّق عند الحاجة.
