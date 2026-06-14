# انحرافات web ↔ partners (Drift) — الجهات والأدوار

هذا الملف يسجّل **الفروق الجوهرية** بين ما يملكه `okta-web` (مصدر الحقيقة للجهات
والأدوار) وما يعكسه/يعلنه `okta-partners`، **والتي تحتاج تكاملًا**. هو امتداد لـ
[`partner-apps-and-roles.md`](partner-apps-and-roles.md): ذاك يصف العقد السليم،
وهذا يرصد أين ينحرف الجانبان اليوم.

> القاعدة الذهبية تبقى: **okta-web مصدر الحقيقة**؛ كل انحراف أدناه = نقطة يجب أن
> يستهلك فيها partners كتالوجًا من web بدل قائمة محلية، أو أن يُشتقّ كتالوج web من
> مصدره الكنسي بدل نسخة يدوية.

> دليل الحالة: 🔴 انحراف يكسر المطابقة · 🟠 لا مصدر مشترك (خطر انحراف) · ⚠️ مزامنة
> ناقصة.

---

## 1) كتالوج أنواع الجهات — مفتاح «شركة تعليمية» — 🔴

| | okta-web | okta-partners |
|---|---|---|
| المصدر الكنسي | `App\Enums\EntityType` — مفتاح **`education_company`** (+ قيد CHECK على `tenants.type`) | مرآة `partner_country_catalog` (+ defaults) — مفتاح **`educational_company`** |

**المنبع داخل web نفسه:** الدالة التي تغذّي الجسر
(`App\Services\PartnerCountries\Catalog\GetCountryCatalog::canonicalTenantTypes()`،
المكشوفة على `GET /api/partners/countries/catalog`) هي **نسخة يدوية ثانية لا تُشتقّ
من `EntityType`**، فتُصدِّر `educational_company`. لذا مرآة partners «متزامنة
بأمانة من مصدرٍ منحرف».

**الأثر:** أي استهداف/تسعير لتطبيق على نوع `educational_company` **لن يطابق أي جهة
فعلية أبدًا** (قيمة الجهة الحقيقية `education_company`).

**التكامل المطلوب:**
- اشتقاق `GetCountryCatalog::canonicalTenantTypes()` من `EntityType` (إنهاء قائمة
  يدوية رابعة بعد التي عولِجت في
  [`entities-tenancy.md`](entities-tenancy.md#2-نوع-الكيان-type)) + اختبار تطابق.
- data-migration في partners لمواءمة الصفوف المخزّنة
  (`partner_supported_tenant_types` + جداول التسعير) أو alias مرحلي
  `educational_company → education_company`.

---

## 2) كتالوج أدوار/جماهير التطبيق (audiences) — لا مصدر مشترك — 🟠

| | okta-web | okta-partners |
|---|---|---|
| الأدوار المتاحة | المزروع كنسيًّا: `tenant-admin` (+ `student`/`guardian` بنطاق `general`)؛ «معلم» = `TenantEmployee.type` **لا دور spatie** | `config/account-types.php` **ساكن**: `tenant-admin, teacher, employee, principal, supervisor, accountant` + custom + بوابتا `student`/`guardian` |

**الأثر:** partners يَعِد المطوّر بظهور تطبيقه لأدوار (`principal`/`supervisor`/
`accountant`/`teacher`…) قد **لا تكون أدوارًا كنسية موجودة** في web؛ المطابقة وقت
التشغيل **بالاسم النصّي** لدور جهةٍ ديناميكي قد لا يُنشئه أحد. ولا تحقّق عند النشر.

**التكامل المطلوب:**
- كتالوج «أنواع حسابات/أدوار» يُكشَف من web عبر الجسر (نمط النطاقات:
  catalog + hash + webhook) يستبدل الـ config الساكن، **أو** — كحدّ أدنى — تحقّق
  publish-time أنّ كل `audience.role` معروف، مع إبقاء `custom_role` معلَّمًا
  «غير مضمون».

---

## 3) مزامنة كتالوج الدول/الأنواع بلا webhook — ⚠️

النطاقات (scopes) تُبَثّ فوريًّا عبر webhook `partner_scopes.catalog.changed`. أما
كتالوج الدول/الأنواع فيصل partners فقط عبر cron/يدوي
(`partners:sync-country-catalog`) — نافذة انحراف بين تعديل web وظهوره للشركاء.

**التكامل المطلوب:** حدث `partners.country_catalog.changed` على الـ webhook الموحّد
(`OktaWebWebhookController`) يستدعي `SyncCountryCatalogFromOktaWeb(force=true)`.

---

## 4) كود يتيم في partners — تنظيف

`App\Services\AppStorePricing\TenantTypes\GetCanonicalTenantTypes` (صاحب الـ
defaults المنحرفة في البند 1) **لا يستدعيه أحد**؛ المستخدَم فعليًّا هو
`AppStorePricing\Countries\GetCountryCatalog`. يُحذف أو يُوحَّد كي لا يلتقطه كود
مستقبلي ويعيد إدخال الانحراف.

---

## جدول موجز

| # | الفرق | web | partners | الحالة | التكامل |
|---|---|---|---|---|---|
| 1 | مفتاح «شركة تعليمية» | `education_company` (enum + CHECK) | `educational_company` | 🔴 | اشتقاق كتالوج web من `EntityType` + مواءمة partners |
| 2 | كتالوج أدوار audiences | لا كتالوج أدوار مكشوف؛ tenant-admin فقط مزروع | config ساكن بأدوار غير مضمونة | 🟠 | كشف عبر الجسر أو تحقّق publish-time |
| 3 | مزامنة الدول/الأنواع | — | cron/يدوي بلا webhook | ⚠️ | webhook `country_catalog.changed` |
| 4 | `GetCanonicalTenantTypes` يتيم | — | غير مُستدعى | تنظيف | حذف/توحيد |

---

## مراجع
- العقد السليم: [`partner-apps-and-roles.md`](partner-apps-and-roles.md)
- كتالوج الأنواع على web: [`entities-tenancy.md`](entities-tenancy.md#2-نوع-الكيان-type) · [`roles-rbac.md`](roles-rbac.md)
- منصّة الشركاء: [`claude/partners.md`](../../../claude/partners.md)
