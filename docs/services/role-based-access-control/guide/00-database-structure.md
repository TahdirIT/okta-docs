# Database structure (Laravel 12 + PostgreSQL)

هذا القسم يوضح **هيكل قاعدة البيانات** لخدمة **التحكم في الوصول القائم على الأدوار** مع افتراض:

- **Laravel 12**
- **PostgreSQL**
- **spatie/laravel-permission** كحزمة أساسية
- **spatie/laravel-multitenancy** (اختياري) أو نظام مستأجر مخصص

> الهدف: توفير نظام مرن لإدارة الصلاحيات والأدوار مع دعم البيئة متعددة المستأجرين وفصل واضح بين الأدوار على مستوى النظام ومستوى المستأجر.

## الجداول الأساسية

### 1) `permissions`

جدول الصلاحيات الأساسي من حزمة `spatie/laravel-permission` مع إضافات:

**الأعمدة الأساسية من الحزمة:**
- `id`: `bigint` (PK)
- `name`: `varchar` (اسم الصلاحية، مثل: `users.create`)
- `guard_name`: `varchar` (الحارس، عادة `web`)
- `created_at` / `updated_at`: `timestamptz`

**الأعمدة الإضافية المضافة:**
- `scope`: `varchar` (`system` | `tenant`) - نطاق الصلاحية
- `tenant_id`: `bigint` nullable (FK إلى `tenants`) - للمستأجر المرتبط بالصلاحية
- `title_ar`: `varchar` nullable - عنوان الصلاحية بالعربية للعرض
- `title_en`: `varchar` nullable - عنوان الصلاحية بالإنجليزية للعرض
- `deleted_at`: `timestamptz` nullable (Soft Deletes)

**القيود:**
- `unique(['name', 'guard_name', 'scope', 'tenant_id'])` - يضمن عدم تكرار الصلاحية في نفس النطاق والمستأجر
- `index(['scope'])`
- `index(['tenant_id'])`

### 2) `roles`

جدول الأدوار الأساسي من حزمة `spatie/laravel-permission` مع إضافات:

**الأعمدة الأساسية من الحزمة:**
- `id`: `bigint` (PK)
- `name`: `varchar` (اسم الدور، مثل: `superadmin`, `admin`, `teacher`)
- `guard_name`: `varchar` (الحارس، عادة `web`)
- `team_id`: `bigint` nullable (من ميزة Teams في الحزمة - يمثل `tenant_id`)
- `created_at` / `updated_at`: `timestamptz`

**الأعمدة الإضافية المضافة:**
- `scope`: `varchar` (`system` | `tenant`) - نطاق الدور
- `tenant_id`: `bigint` nullable (FK إلى `tenants`) - للمستأجر المرتبط بالدور
- `title_ar`: `varchar` nullable - عنوان الدور بالعربية للعرض
- `title_en`: `varchar` nullable - عنوان الدور بالإنجليزية للعرض
- `deleted_at`: `timestamptz` nullable (Soft Deletes)

**القيود:**
- `unique(['tenant_id', 'name', 'guard_name'])` - يضمن عدم تكرار الدور في نفس المستأجر
- `index(['scope'])`
- `index(['tenant_id'])`

### 3) `model_has_permissions`

جدول pivot يربط النماذج (Users) بالصلاحيات (من حزمة `spatie/laravel-permission`):

**الأعمدة:**
- `permission_id`: `bigint` (FK إلى `permissions`)
- `model_type`: `varchar` (مثل: `App\Models\User`)
- `model_id`: `bigint` (ID النموذج)
- `team_id`: `bigint` nullable (من ميزة Teams - يمثل `tenant_id`)

**القيود:**
- Primary key: `['team_id', 'permission_id', 'model_id', 'model_type']` (عند تفعيل Teams)
- أو: `['permission_id', 'model_id', 'model_type']` (بدون Teams)

### 4) `model_has_roles`

جدول pivot يربط النماذج (Users) بالأدوار (من حزمة `spatie/laravel-permission`):

**الأعمدة:**
- `role_id`: `bigint` (FK إلى `roles`)
- `model_type`: `varchar` (مثل: `App\Models\User`)
- `model_id`: `bigint` (ID النموذج)
- `team_id`: `bigint` nullable (من ميزة Teams - يمثل `tenant_id`)
- `tenant_id`: `bigint` nullable (FK إلى `tenants`) - للمستأجر المرتبط بالدور (للأدوار على مستوى المستأجر)
- `deleted_at`: `timestamptz` nullable (Soft Deletes)

**القيود:**
- Primary key: `['team_id', 'role_id', 'model_id', 'model_type']` (عند تفعيل Teams)
- أو: `['role_id', 'model_id', 'model_type']` (بدون Teams)
- `index(['tenant_id'])` - للاستعلامات السريعة على الأدوار حسب المستأجر
- `index(['deleted_at'])` - للاستعلامات السريعة على السجلات المحذوفة
- `tenant_id` يكون `null` للأدوار على مستوى النظام
- `tenant_id` يكون غير `null` للأدوار على مستوى المستأجر

### 5) `role_has_permissions`

جدول pivot يربط الأدوار بالصلاحيات (من حزمة `spatie/laravel-permission`):

**الأعمدة:**
- `permission_id`: `bigint` (FK إلى `permissions`)
- `role_id`: `bigint` (FK إلى `roles`)

**القيود:**
- Primary key: `['permission_id', 'role_id']`

## الأدوار الثابتة (Fixed Roles)

### على مستوى النظام (System Scope)
- `superadmin` - المدير العام للنظام، يملك جميع صلاحيات `scope = system`
- `platform-admin` - مدير المنصة، يملك صلاحيات `rbac.*` + `tenants.*`

### على مستوى المستأجر (Tenant Scope)
- `tenant-admin` - مسؤول الكيان التعليمي، يدير الأعضاء والأدوار داخل المستأجر
- `reviewer` - مراجع، يملك صلاحيات `*.view` فقط
- `member` - عضو عادي، يملك `users.view` فقط

> جميع الأدوار القوالب تُخزَّن بـ `tenant_id = null`. التعيين الفعلي للمستخدم يتم عبر `model_has_roles.tenant_id` (ميزة Teams).

## العلاقات

### Permission Model
- `belongsTo(Tenant::class)` - عبر `tenant_id`

### Role Model
- `belongsTo(Tenant::class)` - عبر `tenant_id`
- `belongsToMany(Permission::class)` - عبر `role_has_permissions`

### User Model
- `morphToMany(Role::class, 'model', 'model_has_roles')` - للأدوار (على مستوى النظام والمستأجر)
  - للأدوار على مستوى النظام: `tenant_id` يكون `null`
  - للأدوار على مستوى المستأجر: `tenant_id` يحتوي على معرف المستأجر
- `hasMany(TenantUser::class)` - للروابط مع المستأجرين

## ملاحظات مهمة

1. **Teams Feature**: عند تفعيل `teams => true` في `config/permission.php`، يتم استخدام `team_id` في الجداول pivot لتمثيل المستأجر.

2. **Middleware Integration**: يتم إدارة `team_id` عبر Middleware عند تغيير سياق المستأجر.

3. **Cache Management**: عند تغيير سياق المستأجر، يجب مسح ذاكرة التخزين المؤقت للصلاحيات.

4. **Unique Constraints**: يجب مراعاة `tenant_id` في القيود الفريدة للصلاحيات والأدوار على مستوى المستأجر.
