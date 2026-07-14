# الطبقة التقنية — CRM

- [نماذج البيانات](./models/README.md) — جداول `crm_*` (حسابات، عملاء محتملون،
  جهات اتصال، أنابيب، أنشطة، مهام، عقود، فواتير، تقويم، مستندات، سجلات وقت).
- [وظائف الخدمة (Use-cases)](./service-functions/README.md) — التقويم، العقود،
  المجلّدات، المستندات، سجلات الوقت، صندوق الوارد.

## الوصول والسياق

- **Middleware `crm`** (`App\Http\Middleware\CrmAccess`): يسمح فقط لمن يحمل أحد
  أدوار CRM الخمسة (أو `superadmin`)، ويُجبر سياق الفريق على `null` — فالـ CRM
  سطح منصّة لا سطح جهة.
- **الصلاحيات على مستوى العنصر** لا تُفرَض في ملف المسار بل داخل كل مكوّن Livewire
  عبر `abort_unless($user->can('leads.view'), 403)` ونظائرها.
- نماذج CRM تمتد `App\Models\Support\BaseModel` فتعرض **hashids** في الروابط بدل
  المعرّفات الرقمية.

## المسارات

- `routes/crm.php` — مجموعة `prefix('crm')->middleware(['auth','crm'])` تغطّي
  لوحة التحكم، العملاء المحتملين، الحسابات، جهات الاتصال، المهام، التقويم، سجلات
  الوقت، العقود وقوالبها، المستندات، الفواتير، وإعدادات الأنبوب/الأدوار.
- تغذية ICS خارج المجموعة (بلا auth، محميّة برمز):
  `/crm/calendar/feed/{user}/{token}.ics`.
- `routes/inbox.php` — `/crm/inbox` بـ `middleware(['auth','crm','permission:crm.inbox.view'])`.

## الصلاحيات

- **صلاحيات العناصر — صيغة `<module>.<action>`** (عبر `CrmRolesSeeder`، نطاق
  system، guard `web`): الوحدات `leads, accounts, contacts, tasks, contracts,
  invoices, documents, calendar, contract_templates, time_tracking` × الأفعال
  `view, create, edit, delete, export` + `pipeline_settings.manage` و
  `role_settings.manage`. خمسة أدوار مبذورة: `crm_account_manager` (كامل)،
  `crm_sales`، `crm_marketing`، `crm_technical_support`، `crm_technical_engineer`.
- **صلاحيات صندوق الوارد — صيغة `crm.inbox.<action>`** (عبر `PermissionSeeder`،
  نطاق system): `crm.inbox.{view,reply,assign,tag,delete_message,manage_settings}`.

## صندوق الوارد (Inbox) — مضمَّن

`GET /crm/inbox` → `Crm\Inbox\InboxFrame` يعرض iframe ملء الارتفاع إلى
okta-whatsapp. `IssueEmbedToken` يُصدر JWT (HS256، `aud=okta-whatsapp`،
`scope=platform.admin`، TTL 300s) بسرّ مشترك من `BridgeSettings::embedSsoSecret()`.
`Crm\Inbox\UnreadBadge` يلتقط `postMessage({type:"inbox.unread",count})` ويعيد بثّه
كحدث Livewire `inbox-unread-updated`. لا جداول/نماذج inbox محليّة — السطح كامله
هو الـ iframe البعيد.
