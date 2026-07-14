# تكامل web ↔ partners — المُوحَّد والفجوات (الجهات والأدوار)

`okta-web` هو **مصدر الحقيقة** لكتالوج الجهات والأدوار، و`okta-partners` **يعكسه**.
هذا الملف يصف — لطبقة الجهات/الأدوار تحديدًا — **ما هو مُوحَّد بين الطرفين اليوم**
و**أين يبقى التوحيد ناقصًا** (فجوة تحتاج مصدرًا مشتركًا أو تحقّقًا). هو امتداد
لـ[`partner-apps-and-roles.md`](partner-apps-and-roles.md) الذي يصف العقد السليم.

> دليل الحالة: 🟠 لا مصدر مشترك (خطر انحراف) · ⚠️ مزامنة ناقصة.

---

## المُوحَّد اليوم

- **كتالوج النطاقات (Scopes)**: `okta-web` مصدره الكنسي (`RegisterScope`)،
  و`okta-partners` يعكسه عبر `SyncFromOktaWeb` إلى `partner_available_scopes`.
  المزامنة hash-aware وتُبَثّ فوريًّا عبر webhook `partner_scopes.catalog.changed`.
- **مفتاح نوع الجهة «شركة تعليمية»**: الطرفان يستخدمان المفتاح الكنسي
  **`education_company`**. web يشتقّه من `App\Enums\EntityType` (+ قيد `CHECK` على
  `tenants.type`)، وpartners يحمل enum محليًّا مطابقًا + مرآة `partner_country_catalog`؛
  والتطابق محروسٌ باختبارات على الجانبين.

---

## الفجوات الحالية

### 1) كتالوج أدوار/جماهير التطبيق (audiences) — لا مصدر مشترك — 🟠

| | okta-web | okta-partners |
|---|---|---|
| الأدوار المتاحة | المزروع كنسيًّا: `tenant-admin` (+ `student`/`guardian` بنطاق `general`)؛ «معلم» = `TenantEmployee.type` **لا دور spatie** | `config/account-types.php` **ساكن**: `tenant-admin, teacher, employee, principal, supervisor, accountant` + custom + بوابتا `student`/`guardian` |

partners يَعِد المطوّر بظهور تطبيقه لأدوار (`principal`/`supervisor`/`accountant`/
`teacher`…) قد **لا تكون أدوارًا كنسية موجودة** في web؛ المطابقة وقت التشغيل تتمّ
**بالاسم النصّي** لدور جهةٍ ديناميكي قد لا يُنشئه أحد، ولا تحقّق عند النشر.

**التكامل المطلوب:** كشف كتالوج «أنواع حسابات/أدوار» من web عبر الجسر (نمط
النطاقات: catalog + hash + webhook) بدل الـ config الساكن، **أو** — كحدّ أدنى —
تحقّق publish-time أنّ كل `audience.role` معروف، مع وسم `custom_role` «غير مضمون».

### 2) مزامنة كتالوج الدول/الأنواع بلا webhook — ⚠️

النطاقات تُبَثّ فوريًّا عبر webhook، أمّا كتالوج الدول/أنواع الجهات فيصل partners
فقط عبر cron/يدوي (`partners:sync-country-catalog`) — نافذة انحراف بين تعديل web
وظهوره للشركاء.

**التكامل المطلوب:** حدث `partners.country_catalog.changed` على الـ webhook الموحّد
(`OktaWebWebhookController`) يستدعي `SyncCountryCatalogFromOktaWeb(force=true)`.

---

## جدول موجز

| الفرق | web | partners | الحالة | التكامل المطلوب |
|---|---|---|---|---|
| كتالوج أدوار audiences | لا كتالوج أدوار مكشوف؛ `tenant-admin` فقط مزروع | config ساكن بأدوار غير مضمونة | 🟠 | كشف عبر الجسر أو تحقّق publish-time |
| مزامنة الدول/الأنواع | مصدر كنسي (`EntityType`) | cron/يدوي بلا webhook | ⚠️ | webhook `country_catalog.changed` |

---

## مراجع

- العقد السليم: [`partner-apps-and-roles.md`](partner-apps-and-roles.md)
- كتالوج الأنواع على web: [`entities-tenancy.md`](entities-tenancy.md) · [`roles-rbac.md`](roles-rbac.md)
- منصّة الشركاء: [`claude/partners.md`](../../../claude/partners.md)
