# `permissions`

## الغرض

تعريف الصلاحيات في النظام. كل صلاحية تمثل إجراءاً محدداً يمكن للمستخدم تنفيذه.

> **ملاحظة مهمة:** هذا الجدول يعتمد على حزمة `spatie/laravel-permission` التي توفر البنية الأساسية. يتم إضافة الحقول الإضافية المطلوبة للمشروع.

## الحزمة المستخدمة

**`spatie/laravel-permission`** - حزمة Laravel شائعة لإدارة الصلاحيات والأدوار.

### الأعمدة من الحزمة

- **id**: `bigint` (PK)
- **name**: `varchar` (اسم الصلاحية، مثل: `users.create`, `settings.permissions.view`)
- **guard_name**: `varchar` (الحارس، عادة `web`)
- **created_at** / **updated_at**: `timestamptz`

### الأعمدة الإضافية المضافة للمشروع

- **scope**: `varchar` (`system` | `tenant`، الافتراضي `tenant`) - نطاق الصلاحية
- **title_ar**: `varchar` nullable - عنوان الصلاحية بالعربية للعرض
- **title_en**: `varchar` nullable - عنوان الصلاحية بالإنجليزية للعرض
- **deleted_at**: `timestamptz` nullable (Soft Deletes)

> **مهم:** جدول `permissions` **لا يحمل عمود `tenant_id`** — التعريف عالمي
> ومقيَّد بالـ `scope` فقط. مفتاح المستأجر (`tenant_id`، ميزة Teams) يعيش على
> جداول الربط `model_has_roles` / `model_has_permissions`، لا على `permissions`.

## العلاقات

- **belongsToMany**: `Role` (via `role_has_permissions`)
- **morphToMany**: `User` (via `model_has_permissions`)

## الفهارس/القيود

- **unique**: (`name`, `guard_name`, `scope`) - القيد الفعلي في okta-web (migration `extend_permissions_table_for_scope`).
- **index**: (`scope`)
- **index**: (`deleted_at`)

## القواعد

### قواعد التحقق

1. **اسم الصلاحية:**
   - مطلوب
   - يجب أن يكون فريداً ضمن نفس النطاق والمستأجر
   - الحد الأقصى للطول: 120 حرف

2. **النطاق:**
   - مطلوب
   - يجب أن يكون `system` أو `tenant`

3. **المستأجر:**
   - للصلاحيات على مستوى المستأجر: مطلوب
   - للصلاحيات على مستوى النظام: يجب أن يكون `null`

### القيود

- الصلاحيات على مستوى النظام لا يمكن ربطها بمستأجر
- الصلاحيات على مستوى المستأجر يجب أن تكون مرتبطة بمستأجر
- لا يمكن حذف صلاحية مرتبطة بدور نشط (Soft Delete)

## الاستخدام

### إنشاء صلاحية

```php
use App\Models\Permission;

// صلاحية على مستوى النظام
Permission::create([
    'name' => 'users.create',
    'guard_name' => 'web',
    'scope' => 'system',
    'tenant_id' => null,
    'title_ar' => 'إنشاء مستخدم',
    'title_en' => 'Create User',
]);

// صلاحية على مستوى المستأجر
Permission::create([
    'name' => 'students.view',
    'guard_name' => 'web',
    'scope' => 'tenant',
    'tenant_id' => $tenantId,
    'title_ar' => 'عرض الطلاب',
    'title_en' => 'View Students',
]);
```

### الاستعلام عن الصلاحيات

```php
// جميع الصلاحيات على مستوى النظام
$systemPermissions = Permission::where('scope', 'system')->get();

// جميع الصلاحيات على مستوى المستأجر
$tenantPermissions = Permission::where('scope', 'tenant')
    ->where('tenant_id', $tenantId)
    ->get();
```

### التحقق من الصلاحية

```php
// في Controller أو Middleware
if ($user->hasPermissionTo('users.create')) {
    // المستخدم لديه الصلاحية
}
```

## التكامل مع spatie/laravel-permission

- يستخدم النموذج `Spatie\Permission\Models\Permission` كأساس
- يتم تخصيص النموذج عبر `App\Models\Permission`
- يتم استخدام ميزة Teams في الحزمة لتمثيل المستأجرين (عند تفعيلها)
