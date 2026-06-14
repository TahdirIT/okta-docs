# انحرافات web ↔ partners (Drift) — الجهات والأدوار

هذا الملف يسجّل **الفروق الجوهرية** بين ما يملكه `okta-web` (مصدر الحقيقة للجهات
والأدوار) وما يعكسه/يعلنه `okta-partners`، **والتي تحتاج تكاملًا**. هو امتداد لـ
[`partner-apps-and-roles.md`](partner-apps-and-roles.md): ذاك يصف العقد السليم،
وهذا يرصد أين ينحرف الجانبان اليوم.

> القاعدة الذهبية تبقى: **okta-web مصدر الحقيقة**؛ كل انحراف أدناه = نقطة يجب أن
> يستهلك فيها partners كتالوجًا من web بدل قائمة محلية، أو أن يُشتقّ كتالوج web من
> مصدره الكنسي بدل نسخة يدوية.

> دليل الحالة: ✅ مُعالَج · 🔴 انحراف يكسر المطابقة · 🟠 لا مصدر مشترك (خطر انحراف)
> · ⚠️ مزامنة ناقصة.

---

## 1) كتالوج أنواع الجهات — مفتاح «شركة تعليمية» — ✅ مُعالَج

| | okta-web | okta-partners |
|---|---|---|
| المصدر الكنسي | `App\Enums\EntityType` — مفتاح **`education_company`** (+ قيد CHECK على `tenants.type`) | `App\Enums\EntityType` محلي (مرآة من web) + مرآة `partner_country_catalog` المُزامَنة — مفتاح **`education_company`** |

**الانحراف الأصلي:** كانت الدالة التي تغذّي الجسر
(`App\Services\PartnerCountries\Catalog\GetCountryCatalog::canonicalTenantTypes()`،
المكشوفة على `GET /api/partners/countries/catalog`) **نسخة يدوية ثانية لا تُشتقّ
من `EntityType`**، فتُصدِّر `educational_company`؛ ومرآة partners كانت «متزامنة
بأمانة من مصدرٍ منحرف». الأثر كان: أي استهداف/تسعير على `educational_company` لا
يطابق أي جهة فعلية (قيمتها الحقيقية `education_company`).

**ما طُبِّق:**
- **web** — `GetCountryCatalog::canonicalTenantTypes()` صار يشتقّ المفاتيح من
  `EntityType::cases()` (التسميات ثنائية اللغة عبر `match` بلا `default`، فأي حالة
  enum جديدة بلا تسمية تُسقِط الاختبار). يحرسه
  `tests/Feature/PartnerBridge/CountryCatalogTenantTypesTest.php`.
- **partners** — استُنسِخ `App\Enums\EntityType` محليًّا بالمفتاح الكنسي
  `education_company`، واشتُقّت منه `AppStorePricing\Countries\GetCountryCatalog`
  defaults، وحُذف الكود اليتيم `GetCanonicalTenantTypes`. يحرسه
  `tests/Feature/AppStorePricing/EntityTypeCatalogTest.php`.
- **مواءمة البيانات** — المرآة `partner_country_catalog` تُصحَّح ذاتيًّا عند أوّل
  مزامنة بعد إصلاح الجسر (تغيّر الكتالوج ⇒ تغيّر الـ hash ⇒ إعادة سحب). والصفوف
  المحفوظة التي لا تلمسها المزامنة صحّحها migration
  `2026_06_14_000000_normalize_educational_company_tenant_type_code`
  (`partner_supported_tenant_types` + `partner_module_pricings` +
  `partner_payouts`).

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

## 4) كود يتيم في partners — ✅ مُعالَج (حُذف)

كان `App\Services\AppStorePricing\TenantTypes\GetCanonicalTenantTypes` (صاحب الـ
defaults المنحرفة في البند 1) **لا يستدعيه أحد**، والمستخدَم فعليًّا هو
`AppStorePricing\Countries\GetCountryCatalog`. حُذف ضمن إصلاح البند 1 كي لا يلتقطه
كود مستقبلي ويعيد إدخال الانحراف؛ صار `GetCountryCatalog` يشتقّ من
`App\Enums\EntityType` المحلي.

---

## جدول موجز

| # | الفرق | web | partners | الحالة | التكامل |
|---|---|---|---|---|---|
| 1 | مفتاح «شركة تعليمية» | `education_company` (enum + CHECK) | `education_company` (enum محلي + مرآة) | ✅ | اشتُقّ كتالوج web من `EntityType` + enum محلي في partners + migration مواءمة |
| 2 | كتالوج أدوار audiences | لا كتالوج أدوار مكشوف؛ tenant-admin فقط مزروع | config ساكن بأدوار غير مضمونة | 🟠 | كشف عبر الجسر أو تحقّق publish-time |
| 3 | مزامنة الدول/الأنواع | — | cron/يدوي بلا webhook | ⚠️ | webhook `country_catalog.changed` |
| 4 | `GetCanonicalTenantTypes` يتيم | — | حُذف | ✅ | حُذف ضمن إصلاح البند 1 |

---

## مراجع
- العقد السليم: [`partner-apps-and-roles.md`](partner-apps-and-roles.md)
- كتالوج الأنواع على web: [`entities-tenancy.md`](entities-tenancy.md) · [`roles-rbac.md`](roles-rbac.md)
- منصّة الشركاء: [`claude/partners.md`](../../../claude/partners.md)
