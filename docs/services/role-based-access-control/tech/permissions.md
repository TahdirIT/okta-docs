# صلاحيات خدمة RBAC (كما هي مبذورة في okta-web)

أكواد الصلاحيات الفعلية لخدمة **التحكم في الوصول** كما يبذرها
`database/seeders/PermissionSeeder.php`. الصيغة تتبع معيار المنصة
`<feature>.<resource>.<action>` (راجع
[`tech-standards/permissions-naming.md`](../../../tech-standards/permissions-naming.md)):
**بلا wildcards** (`*`)، وبأفعال محددة.

> الوصول الشامل يأتي من **الدور** (superadmin يملك كل صلاحيات `scope=system`)،
> لا من مفتاح صلاحية wildcard — لا توجد مفاتيح `*` في أسماء الصلاحيات.

## نطاق النظام (System) — إدارة RBAC المركزية

- `rbac.permissions.view` · `rbac.permissions.create` · `rbac.permissions.delete`
- `rbac.roles.view` · `rbac.roles.create` · `rbac.roles.update` · `rbac.roles.delete`
- `rbac.users.view` · `rbac.users.create` · `rbac.users.update` · `rbac.users.delete` · `rbac.users.merge`

## نطاق المستأجر (Tenant) — إدارة الأدوار/المستخدمين داخل الجهة

- `roles.view` · `roles.create` · `roles.update` · `roles.delete` · `roles.assign` · `roles.revoke`
- `users.view` · `users.create` · `users.update` · `users.delete`

> يملك `tenant-admin` هذه الصلاحيات (وغيرها) عبر أنماط `RoleSeeder`
> (`users.%` + قائمة `roles.*` صريحة). راجع
> [بنية قاعدة البيانات](../guide/00-database-structure.md#الأدوار-الثابتة-fixedseeded-roles).

## القواعد

- التحقق من عدم تكرار اسم الصلاحية داخل نفس `scope` (القيد
  `unique(name, guard_name, scope)`).
- صلاحيات `scope=system` لا تُربَط بمستأجر؛ صلاحيات `scope=tenant` تُقيَّم ضمن
  `team_id = tenant_id`.
- لا تُحذف صلاحية مرتبطة بدور نشط.
- تُسجَّل عمليات الإنشاء/التعديل/الحذف في سجل التدقيق.

> **ملاحظة عن `manage`:** المعيار يثنّي عن `manage`، ومع ذلك يستخدمه الكود في حالات
> قليلة محددة خارج RBAC (مثل `tenant.profile.manage`،
> `notifications.providers.manage`، `payments.providers.manage`) — ليست جزءاً من
> صلاحيات RBAC أعلاه.
