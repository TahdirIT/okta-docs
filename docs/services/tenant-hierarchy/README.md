# كونسول الجهات الحاوية (Tenant Hierarchy)

الجهات الحاوية (`complex`, `education_company`, `university` —
`EntityType::isContainer()`) تدير جهاتها التابعة عبر شجرة `tenants.parent_tenant_id`.
هذا يوثّق **الكونسول وخدماته**؛ أنواع الكيانات نفسها في
[`roles-and-entities/`](../../roles-and-entities/README.md).

## الخدمات (`app/Services/Tenants/Hierarchy/*`)

- `GetChildrenWithStats` — الأبناء المباشرون + إحصاءات (طلاب/موظفون/أبناء)، cached 60s.
- `GetTreeSummary` — إجماليات الشجرة كاملة، cached 60s.
- `AccessChildTenant` — «دخول» أدمن الحاوية لجهة ضمن شجرته: يفحص `tenants.children.access`،
  يجسّد العضوية (`tenant_user` + دور `tenant-admin` مقيّد)، يسجّل `hierarchy.child_access_granted`،
  ثم يبدّل سياق الجلسة.
- `GetTreeStudents` / `GetTreeEmployees` — Builders عبر كل جهات الشجرة (`withoutTenantScope`) بفلاتر.
- `GetEntityComparison` — صف مقارنة لكل ابن مباشر (طلاب/موظفون/صفوف/شعب/نسبة)، cached 60s.
- `BulkInstallModuleForChildren` — تثبيت تطبيق متجر جماعياً (عزل فشل كل جهة، سجل `hierarchy.bulk_app_install`).
- `PushAcademicStructure` — تعميم الصفوف/الشعب/المواد من مصدر إلى تابعات (idempotent).
- `BroadcastToChildren` — رسالة جماعية لمنسوبي التابعات (عبر `Messaging\Messages\SendMessage`).
- `CreateChildTenant` — إنشاء جهة تابعة ذاتياً (`parent_tenant_id`، بلا اشتراك).

## الصفحات (`App\Livewire\Tenant\Children\*` — قسم «الجهات التابعة»)

| المسار | الصفحة | القدرة |
|---|---|---|
| `/tenant/children` | `ChildrenIndex` — بطاقات + جدول أبناء + «دخول» | `tenants.children.view` |
| `/tenant/children/students` · `/employees` | سجل موحّد عبر الشجرة + تصدير Excel | `view_members` |
| `/tenant/children/reports/comparison` | جدول مقارنة + تصدير Excel/PDF | `view_reports` |
| `/tenant/children/apps` | تثبيت تطبيقات جماعي | `install_apps` + ميزة `hierarchy.bulk_install` |
| `/tenant/children/{child}` | ملف الجهة التابعة + إسناد مدير | `manage_admins` |
| `/tenant/children/academics` | تعميم الهيكل الأكاديمي | `push_academics` |
| `/tenant/children/broadcast` | رسالة جماعية | `broadcast` + ميزة `messaging.broadcast` |
| `/tenant/children/activity` | سجل أحداث `hierarchy.*` | `view_activity` |

## الصلاحيات وميزات الباقة

- **الصلاحيات** (نطاق tenant، تنسال لـ tenant-admin عبر `tenants.children.%`):
  `view, access, view_members, view_reports, create, install_apps, manage_admins,
  push_academics, broadcast, view_activity`.
- **مميزات الباقة** (مجموعة `hierarchy`، كلها default ON): `hierarchy.console`
  (يبوّب الكونسول)، `hierarchy.reports` (requires console)، `hierarchy.bulk_install`
  (requires console) — مفروضة في السايدبار و`mount()` لكل صفحة.

## الاشتراك المركزي (covers_children)

`Tenant::coveringSubscription()` يصعد سلسلة `parent_tenant_id` لأول اشتراك مفعَّل
عليه `covers_children`، و`effectiveSubscription()` = الخاص إن توفّر وإلا التغطية.
`PlanGate::tenantHas` يستخدم هذا الـ fallback (راجع
[الاشتراكات](../subscriptions/README.md#التغطية-المركزية-covers_children)).

- **Exports**: `app/Exports/Tenants/*` (Excel + PDF مقارنة).
- **لوحة `/dashboard`** تعرض واجهات مخصّصة للحاويات وتستهلك نفس الخدمات.
- مفاتيح الترجمة تحت `app.hierarchy.*`.
