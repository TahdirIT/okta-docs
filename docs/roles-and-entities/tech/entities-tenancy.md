# الكيانات وتعدّد المستأجرين (Entities & Tenancy) — تقنيًا

كيف يتحوّل **«نوع الحساب / الكيان»** الموصوف في [`tenants.md`](../tenants.md) إلى
كود داخل `okta-web`، وكيف يُعزَل كل مستأجر عن غيره.

---

## 1) جدول `tenants` ونموذج `Tenant`

الكيان (Tenant) هو الصفّ الجذري لكل جهة مشتركة في المنصّة.

- **Migration**: `database/migrations/2026_02_08_110911_create_tenants_table.php`
- **Model**: `app/Models/Tenant.php` — يمدّد `Spatie\Multitenancy\Models\Tenant`
  (عبر `BaseTenant`)، ويستخدم `HasUlid`, `SoftDeletes`.

أبرز الأعمدة:

| العمود | النوع | الدور |
|---|---|---|
| `id` | bigint PK | المفتاح الداخلي. |
| `type` | string | **نوع الكيان** (انظر القسم 2). |
| `name` / `slug` | string | الاسم + معرّف URL فريد (يُولَّد آليًا). |
| `ulid` | string | معرّف عام ثابت — يُستخدَم كمفتاح ربط للجسر مع الشركاء. |
| `country_id` / `education_level_group_id` | FK | ربط الكيان ببيانات الدولة والمراحل (من خدمة إدارة الدول). |
| `status` | string | `active` \| `suspended` (راجع `Active()` scope). |
| `is_sandbox` | bool | تمييز جهات بيئة الـ sandbox. |
| `owner_partner_tenant_id` | FK nullable | ملكية شريك للجهة (سياق الشركاء)، **ليست** احتواءً تعليميًّا. |
| `registration_custom_fields` | json | لقطة من إجابات معالج التسجيل. |
| `deleted_at` | timestamptz | حذف ناعم. |

> العضوية والأدوار **لا** تعيش هنا؛ المستخدمون يرتبطون بالكيان عبر pivot
> `tenant_user` ونماذج العضوية (راجع [`roles-rbac.md`](roles-rbac.md)).

---

## 2) نوع الكيان (`type`) — قائمة مسطّحة، لا شجرة

نوع الحساب يُخزَّن كـ **string مسطّح** في `tenants.type`. القيم التي يصفها
[`tenants.md`](../tenants.md) تُمثَّل جميعها كقيم لهذا العمود الواحد.

**المصدر الكنسي للأنواع** هو enum `App\Enums\EntityType` (مفاتيح `snake_case` ↔
تسميات عربية)، ويُطابِقه `GetSettings::ENTITY_TYPES` (اختبار وحدة يحرس التطابق):

```php
'individual_teacher' => 'معلم فردي',   'school'           => 'مدرسة',
'kindergarten'       => 'روضة',        'complex'          => 'مجمع',
'college'            => 'كلية',         'university'       => 'جامعة',
'institute'          => 'معهد',         'academy'          => 'أكاديمية',
'education_company'  => 'شركة تعليمية',
```

- **حاجز على مستوى القاعدة**: قيد `CHECK (type IN (...))` على `tenants.type`
  (Postgres؛ مُضاف `NOT VALID` فيفرض على كل INSERT/UPDATE جديد، وعلى الإنتاج
  يُشدَّد على الصفوف القائمة بـ `VALIDATE CONSTRAINT` بعد التدقيق). الاختبارات على
  sqlite تتخطّى القيد (محروس بالـ driver).
- **طبقة التطبيق** أيضًا: `RegisterTenant::validateTenantData()` و`TenantEditorModal`
  يرفضان أي نوع ليس مفتاحًا كنسيًّا. لكل دولة يمكن **تمكين/تعطيل** مجموعة فرعية عبر
  `GetEntityRegistrationCustomization`. ووصول مُنمَّط آمن عبر `Tenant::entityType()`.
- بعض الأنواع تتطلّب اختيار مرحلة تعليمية عند التسجيل — يحدّدها
  `EntityTypesRequiringEducationLevel::KEYS = ['school', 'individual_teacher']`
  (روضة غير مُلزَمة بمرحلة حاليًا).
- **متابعة تنظيف مُعلَّقة**: لا تزال قوائم يدوية مكرّرة (`SubjectModal`،
  بعض الـ presenters/blade، وتعليق الـ migration الأقدم) غير مشتقّة من الـ enum
  وتُسقِط بعض المفاتيح — اعتمد دائمًا `App\Enums\EntityType` / `GetSettings::ENTITY_TYPES`.
- التمييز بين «جهة تشغيلية» و«حاوية» مُرمَّز في `EntityType::isContainer()`،
  والاحتواء الفعلي يُمثَّل بعمود `parent_tenant_id` (انظر [الاحتواء](#الاحتواء)).

### الاحتواء

العلاقة «شركة تعليمية → مجمع → مدرسة/روضة» و«جامعة → كلية» الموصوفة في
[`role-hierarchy.md`](../role-hierarchy.md) **منفَّذة** كقائمة جوار
(adjacency list):

- **العمود**: `tenants.parent_tenant_id` (self-FK، nullable، `nullOnDelete`).
  `null` = جهة عليا/مستقلّة. (migration `2026_06_10_000003_add_parent_tenant_id_to_tenants`.)
- **العلاقات** على `Tenant`: `parent()` / `children()` / `scopeTopLevel()` +
  `descendantIds()` (تُستخدم لمنع الدورات).
- **قواعد النوع** في `App\Enums\EntityType`: `allowedChildTypes()` /
  `isContainer()` / `canContain()` / `containerTypes()` — مثلًا
  `EducationCompany` يحتوي {Complex, University, College, Institute, Academy,
  Kindergarten, School}، و`Complex` يحتوي {School, Kindergarten}، و`University`
  يحتوي {College}؛ والبقية أوراق (لا تحتوي).
- **الربط**: خدمة `App\Services\Tenants\Containment\SetTenantParent` تفرض تطابق
  الأنواع وتمنع self-parent والدورات (الأب ليس من نسل الابن). تُستدعى من
  `TenantEditorModal` — **عملية مسؤول منصّة** (نطاق system).

> ميِّز عن `owner_partner_tenant_id` (ملكية **شريك**، سياق okta-partners) — لا
> علاقة له بالاحتواء التعليمي.

**التجميع للقراءة (مُنفَّذ):** لوحة التحكم (`pages/dashboard`) تعرض لوحة خاصّة لكل
حاوية — المجمع (عدّ المدارس/الروضات + إجمالي طلاب/موظفي الأبناء) والشركة التعليمية
(تجميع الشجرة كاملة عبر `Tenant::descendantIds()` + `Model::withoutTenantScope()`
مقيَّدًا بمعرّفات التابعين، فيتجاوز عزل `BelongsToTenant` بأمان دون `makeCurrent`).

**حدود التنفيذ الحالي (مؤجَّل):**

- **لا تنقّل سياق للكتابة**: مدير الجهة الأم لا يدخل سياق جهة تابعة ليُعدّل بياناتها
  (الأدوار معزولة بـ `team_id` لكل جهة؛ التجميع أعلاه للقراءة فقط).
- الربط **إداري فقط**؛ لا تسجيل ذاتي يدّعي أبًا.

---

## 3) تعدّد المستأجرين (Multitenancy)

المنصّة تستخدم `spatie/laravel-multitenancy 4` بنمط **قاعدة بيانات واحدة
مُفلتَرة حسب المستأجر** (single DB, scoped) — لا قاعدة بيانات منفصلة لكل جهة.

### المستأجر الحالي

- `Tenant::makeCurrent()` يضبط الجهة النشطة للطلب.
- `Tenant::forgetCurrent()` يلغيها (سياق النظام/العام).
- `Tenant::current()` يُرجِع الجهة النشطة.

### الفلترة التلقائية: `BelongsToTenant`

- **Trait**: `app/Models/Concerns/BelongsToTenant.php`.
- يضيف **global scope** يحقن `where tenant_id = Tenant::current()` على كل استعلام،
  ويملأ `tenant_id` تلقائيًا عند الإنشاء.
- للتجاوز المتعمَّد: `Model::withoutTenantScope()`.

### مزامنة سياق الصلاحيات مع المستأجر

عند `makeCurrent()`، تعمل مهمّة ضمن سلسلة مهام spatie:

- **`app/Multitenancy/Tasks/SyncPermissionTeamTask.php`** تستدعي
  `PermissionRegistrar::setPermissionsTeamId($tenant->getKey())`.

أي أنّ **معرّف الـ team في spatie/laravel-permission = `tenant_id`**. هذا هو
الجسر الذي يجعل أدوار المستخدم تُقيَّم ضمن سياق الجهة الحالية تلقائيًا — تفاصيله في
[`roles-rbac.md`](roles-rbac.md).

```mermaid
flowchart LR
  Req["طلب وارد"] --> MC["Tenant::makeCurrent()"]
  MC --> GS["BelongsToTenant\n(global scope: tenant_id=...)"]
  MC --> ST["SyncPermissionTeamTask\nsetPermissionsTeamId(tenant_id)"]
  GS --> Q["استعلامات معزولة للجهة"]
  ST --> P["أدوار/صلاحيات بسياق الجهة"]
```

---

## 4) `PartnerTenant` — مرآة قراءة فقط لقاعدة الشركاء

داخل okta-web يوجد نموذج **`app/Models/PartnerTenant.php`** على اتصال `partners`
(قاعدة بيانات okta-partners، خادم منفصل). هو **مرآة للقراءة فقط** لبيانات الشريك
(المطوّر) — يُستخدَم مثلًا في `Module.partner`.

ميِّز بوضوح:

- **`Tenant`** (في okta-web) = **جهة تعليمية** (مدرسة/جامعة…) ولها طلاب وموظفون
  وأدوار.
- **`PartnerTenant`** = **حساب شريك/مطوّر** يؤلّف التطبيقات. لا يملك طلابًا ولا أدوار
  مستخدم نهائي.

نظير ذلك على الجانب الآخر: جدول `tenants` داخل **okta-partners** يمثّل **الشركاء**،
وله عمود `okta_tenant_ref` للربط بكيان okta-web عند الحاجة (راجع
[`partner-apps-and-roles.md`](partner-apps-and-roles.md)).

---

## 5) تثبيت التطبيقات على الكيان

ربط «جهة ← تطبيق مثبَّت» يعيش في:

- **Migration/Model**: `tenant_module_installations` ←→ `app/Models/TenantModuleInstallation.php`.
- أعمدة: `tenant_id`, `module_id`, `status` (`active`/`inactive`),
  `approved_permissions` (json — النطاقات الممنوحة)، `installed_at`.
- وصول: `$tenant->moduleInstallations()`.

هذا الجدول هو المنطلق لإصدار **رمز التثبيت** (`PartnerInstallationToken`) الذي
يستخدمه التطبيق للوصول لبيانات الجهة ضمن النطاقات الممنوحة — تفاصيله في
[`partner-apps-and-roles.md`](partner-apps-and-roles.md).

---

## مراجع

- المفاهيم: [`tenants.md`](../tenants.md) · [`role-hierarchy.md`](../role-hierarchy.md)
- التسجيل والنماذج: [`tenant-registration/tech`](../../services/tenant-registration/tech/README.md)
- العضوية: [`tenant-members-management/tech`](../../services/tenant-members-management/tech/architecture.md)
