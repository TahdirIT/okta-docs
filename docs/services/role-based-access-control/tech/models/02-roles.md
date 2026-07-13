# `roles`

## الغرض

تعريف الأدوار في النظام. كل دور يمثل مجموعة من الصلاحيات المرتبطة به.

> **ملاحظة مهمة:** هذا الجدول يعتمد على حزمة `spatie/laravel-permission` التي توفر البنية الأساسية. يتم إضافة الحقول الإضافية المطلوبة للمشروع.

## الحزمة المستخدمة

**`spatie/laravel-permission`** - حزمة Laravel شائعة لإدارة الصلاحيات والأدوار.

### الأعمدة من الحزمة

- **id**: `bigint` (PK)
- **name**: `varchar` (اسم الدور، مثل: `superadmin`, `admin`, `teacher`)
- **guard_name**: `varchar` (الحارس، عادة `web`)
- **team_id**: `bigint` nullable (من ميزة Teams في الحزمة - يمثل `tenant_id`)
- **created_at** / **updated_at**: `timestamptz`

### الأعمدة الإضافية المضافة للمشروع

- **scope**: `varchar` (`system` | `tenant`) - نطاق الدور
- **tenant_id**: `bigint` nullable (FK إلى `tenants`) - للمستأجر المرتبط بالدور
- **title_ar**: `varchar` nullable - عنوان الدور بالعربية للعرض
- **title_en**: `varchar` nullable - عنوان الدور بالإنجليزية للعرض
- **deleted_at**: `timestamptz` nullable (Soft Deletes)

## العلاقات

- **belongsTo**: `Tenant` (via `tenant_id`)
- **belongsToMany**: `Permission` (via `role_has_permissions`)
- **morphToMany**: `User` (via `model_has_roles`) - للأدوار (على مستوى النظام والمستأجر)

## الفهارس/القيود

- **unique**: (`tenant_id`, `name`, `guard_name`) - يضمن عدم تكرار الدور في نفس المستأجر
- **index**: (`scope`)
- **index**: (`tenant_id`)
- **index**: (`deleted_at`)

## القواعد

### قواعد التحقق

1. **اسم الدور:**
   - مطلوب
   - يجب أن يكون فريداً ضمن نفس المستأجر
   - الحد الأقصى للطول: 120 حرف

2. **النطاق:**
   - مطلوب
   - يجب أن يكون `system` أو `tenant`

3. **المستأجر:**
   - للأدوار على مستوى المستأجر: مطلوب
   - للأدوار على مستوى النظام: يجب أن يكون `null`

### القيود

- الأدوار على مستوى النظام لا يمكن ربطها بمستأجر
- الأدوار على مستوى المستأجر يجب أن تكون مرتبطة بمستأجر
- لا يمكن حذف الأدوار الثابتة (`superadmin`, `platform-admin`, `tenant-admin`, `reviewer`, `member`)
- لا يمكن حذف دور مرتبط بمستخدمين (Soft Delete)

## الأدوار الثابتة

### على مستوى النظام

- **superadmin**: المدير العام للنظام
  - يملك جميع صلاحيات `scope = system`
  - لا يمكن حذفه
  - يمكن إضافة أدوار جديدة على مستوى النظام

- **platform-admin**: مدير المنصة
  - يملك صلاحيات `rbac.*` + `tenants.*`
  - لا يمكن حذفه

### على مستوى المستأجر

- **tenant-admin**: مسؤول الكيان التعليمي
  - يملك `users.*` + `roles.assign/revoke` + `tenants.members.*`
  - لا يمكن حذفه
  - يمكن إضافة أدوار مخصصة لمستأجره

- **reviewer**: مراجع
  - يملك جميع صلاحيات `*.view` (tenant)
  - لا يمكن حذفه

- **member**: عضو عادي
  - يملك `users.view` فقط
  - لا يمكن حذفه

## الاستخدام

### إنشاء دور

```php
use App\Models\Role;

// دور على مستوى النظام
$role = Role::create([
    'name' => 'content-manager',
    'guard_name' => 'web',
    'scope' => 'system',
    'tenant_id' => null,
    'title_ar' => 'مدير المحتوى',
    'title_en' => 'Content Manager',
]);

// ربط الصلاحيات بالدور
$role->permissions()->sync([$permission1->id, $permission2->id]);

// دور على مستوى المستأجر
$role = Role::create([
    'name' => 'assistant-teacher',
    'guard_name' => 'web',
    'scope' => 'tenant',
    'tenant_id' => $tenantId,
    'title_ar' => 'معلم مساعد',
    'title_en' => 'Assistant Teacher',
]);
```

### الاستعلام عن الأدوار

```php
// جميع الأدوار على مستوى النظام
$systemRoles = Role::where('scope', 'system')->get();

// جميع الأدوار على مستوى المستأجر
$tenantRoles = Role::where('scope', 'tenant')
    ->where('tenant_id', $tenantId)
    ->get();

// الأدوار مع الصلاحيات
$rolesWithPermissions = Role::with('permissions')->get();
```

### تعيين دور لمستخدم

```php
// على مستوى النظام
$user->assignRole('superadmin');
// أو مع تحديد tenant_id صريح
$user->roles()->attach($roleId, ['tenant_id' => null]);

// على مستوى المستأجر
$user->roles()->attach($roleId, ['tenant_id' => $tenantId]);
```

### التحقق من الدور

```php
// في Controller أو Middleware
if ($user->hasRole('superadmin')) {
    // المستخدم لديه الدور
}
```

## التكامل مع spatie/laravel-permission

- يستخدم النموذج `Spatie\Permission\Models\Role` كأساس
- يتم تخصيص النموذج عبر `App\Models\Role`
- يتم استخدام ميزة Teams في الحزمة لتمثيل المستأجرين (عند تفعيلها)
- يتم ربط الأدوار على مستوى المستأجر عبر جدول `model_has_roles` مع تحديد `tenant_id`
