# الأدوار والصلاحيات

## نظرة عامة

يتكون نظام التحكم في الوصول من مكونين رئيسيين: **الأدوار (Roles)** و **الصلاحيات (Permissions)**. الصلاحيات تمثل الإجراءات المحددة التي يمكن للمستخدم تنفيذها، بينما الأدوار تمثل مجموعات من الصلاحيات.

## الصلاحيات (Permissions)

### هيكل الصلاحية

كل صلاحية تحتوي على:
- **الاسم (name)**: معرف فريد للصلاحية (مثل: `users.create`, `rbac.roles.view`)
- **الحارس (guard_name)**: الحارس المستخدم (عادة `web`)
- **النطاق (scope)**: `system` أو `tenant`
- **المستأجر (tenant_id)**: دائماً `null` في الصلاحيات الحالية (القوالب عامة)
- **العناوين (title_ar, title_en)**: عناوين للعرض للمستخدمين

### أنواع الصلاحيات

#### 1. صلاحيات على مستوى النظام (System Scope)
مرتبطة بإدارة النظام ككل، `tenant_id` يكون `null`.

| المجموعة | أمثلة |
|---|---|
| إدارة RBAC | `rbac.permissions.view`, `rbac.roles.create`, `rbac.users.update` |
| إدارة المستأجرين | `tenants.view`, `tenants.create`, `tenants.update`, `tenants.delete` |
| إدارة الدول | `countries_management.countries.view`, ... |

#### 2. صلاحيات على مستوى المستأجر (Tenant Scope)
مرتبطة بالعمليات داخل كيان تعليمي، `tenant_id` يكون `null` (القالب مشترك، التعيين يتم عبر `model_has_roles.tenant_id`).

| المجموعة | أمثلة |
|---|---|
| إدارة الأعضاء | `tenants.members.view`, `tenants.members.assign-roles`, `tenants.members.revoke-roles` |
| إدارة المستخدمين | `users.view`, `users.create`, `users.update`, `users.delete` |
| تعيين الأدوار | `roles.assign`, `roles.revoke` |

### معيار تسمية الصلاحيات

```
<feature>.<resource>.<action>
```

- كل الأجزاء بالحروف الصغيرة
- الفاصل بين الأجزاء نقطة `.`
- لمزيد من التفاصيل راجع [معايير تسمية الصلاحيات](../../../tech-standards/permissions-naming.md)

---

## الأدوار (Roles)

### هيكل الدور

كل دور يحتوي على:
- **الاسم (name)**: معرف فريد للدور
- **الحارس (guard_name)**: الحارس المستخدم (عادة `web`)
- **النطاق (scope)**: `system` أو `tenant`
- **المستأجر (tenant_id)**: `null` للأدوار القوالب العامة، أو `bigint` لأدوار مخصصة لمستأجر
- **العناوين (title_ar, title_en)**: عناوين للعرض للمستخدمين

### الأدوار الثابتة المُنفّذة

#### على مستوى النظام (System Scope)

| الدور | الصلاحيات |
|---|---|
| `superadmin` | جميع صلاحيات `scope = system` |
| `platform-admin` | `rbac.*` + `tenants.*` (system) |

#### على مستوى المستأجر (Tenant Scope)

| الدور | الصلاحيات |
|---|---|
| `tenant-admin` | `users.*` + `roles.assign/revoke` + `tenants.members.*` |
| `reviewer` | جميع صلاحيات `*.view` (tenant) |
| `member` | `users.view` فقط |

> **ملاحظة:** الأدوار القوالب تُخزَّن بـ `tenant_id = null`. عند تعيين دور لمستخدم داخل مستأجر يُسجَّل الـ `tenant_id` في جدول `model_has_roles` عبر ميزة Teams في spatie/laravel-permission.

### الأدوار الديناميكية

- `superadmin` يمكنه إضافة أدوار جديدة على مستوى النظام.
- `tenant-admin` يمكنه إضافة أدوار مخصصة لمستأجره.

---

## ربط المستخدمين بالأدوار

### آلية Teams (spatie/laravel-permission)

يُستخدم حقل `team_id` في جداول pivot (`model_has_roles`, `model_has_permissions`) لتمثيل المستأجر. يتم تعيين هذا الحقل تلقائياً عبر:

```php
app(\Spatie\Permission\PermissionRegistrar::class)->setPermissionsTeamId($tenantId);
```

يُستدعى هذا في Middleware عند تفعيل سياق المستأجر.

### على مستوى النظام

```php
// تعيين دور system مباشرة
$user->assignRole('superadmin');

// التحقق
$user->hasRole('superadmin'); // true
```

### على مستوى المستأجر

```php
// تعيين دور داخل مستأجر عبر HasTenantRbac
app(\Spatie\Permission\PermissionRegistrar::class)->setPermissionsTeamId($tenant->id);
$user->assignRole('tenant-admin');

// التحقق داخل سياق محدد
$user->hasRoleInTenant('tenant-admin', $tenant); // عبر HasTenantRbac trait
$user->hasPermissionInTenant('users.update', $tenant);
```

---

## السياق النشط (Active Context)

السياق يُحفظ في الجلسة ويحتوي على:

```php
session('context') = [
    'scope'            => 'tenant',  // أو 'system'
    'tenant_id'        => 5,
    'tenant_user_id'   => 12,
    'role_ids'         => [3],
    'active_role_name' => 'tenant-admin',
    'active_role_key'  => 'tenant-admin',
];
```

### دورة حياة السياق

1. تسجيل الدخول → توجيه إلى `auth/context` لاختيار الدور.
2. اختيار الدور → حفظ السياق في الجلسة + استدعاء `setPermissionsTeamId`.
3. كل طلب → Middleware يتحقق من صحة السياق ويُحدّث الـ team_id في الـ PermissionRegistrar.
4. تغيير السياق → مسح cache الصلاحيات + تحديث الجلسة.

### Middleware المتعلق بالسياق

| Middleware | الاستخدام |
|---|---|
| `context` | يتحقق من وجود السياق في الجلسة |
| `active-tenant-context` | يتحقق من صحة المستأجر وربط المستخدم به |
| `active-roles` | يتحقق من أن الأدوار النشطة لم تُحذف |
| `active-user` | يتحقق من أن المستخدم غير معلّق |
| `tenant.permission` | يتحقق من صلاحية محددة في السياق الحالي |
| `tenant.role` | يتحقق من دور محدد في السياق الحالي |
