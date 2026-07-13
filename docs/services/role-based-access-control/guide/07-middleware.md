# الوسائط (Middleware)

## نظرة عامة

يستخدم النظام عدة Middleware لإدارة السياق والتحقق من الصلاحيات. تعمل هذه Middleware معاً لضمان أن المستخدم لديه السياق الصحيح والصلاحيات المطلوبة للوصول إلى الصفحات المحددة.

## Middleware المتاحة

### 1. EnsureContext

**الغرض:** التحقق من وجود سياق نشط في الجلسة.

**السلوك:**
- يتحقق من وجود مفتاح `context` في الجلسة
- إذا لم يكن موجوداً، يتم إعادة التوجيه إلى صفحة اختيار السياق (`/auth/context`)
- يتم تنفيذه قبل الوصول إلى الصفحات المحمية

**الاستخدام:**
```php
Route::middleware(['auth', 'context'])->group(function () {
    // Routes that require context
});
```

### 2. EnsureActiveTenantContext

**الغرض:** التحقق من صحة سياق المستأجر النشط.

**السلوك:**
- يتحقق من وجود سياق نشط في الجلسة
- إذا كان السياق على مستوى النظام (`scope === 'system'`)، يتم التجاوز (pass through)
- إذا كان السياق على مستوى المستأجر:
  - يتحقق من وجود `tenant_id` و `tenant_user_id` في السياق
  - يتحقق من أن المستأجر موجود ونشط وغير محذوف
  - يتحقق من أن رابط المستأجر-المستخدم موجود وصالح
  - إذا فشل أي تحقق، يتم إرجاع خطأ 403

**الاستخدام:**
```php
Route::middleware(['auth', 'active-tenant-context'])->group(function () {
    // Routes that require active tenant context
});
```

### 3. EnsureActiveRoles

**الغرض:** التحقق من وجود أدوار نشطة في السياق.

**السلوك:**
- يتحقق من وجود أدوار نشطة في السياق (`role_ids` في الجلسة)
- إذا لم تكن هناك أدوار نشطة، يتم إعادة التوجيه إلى صفحة اختيار السياق
- يتم تنفيذه على الصفحات التي تتطلب أدواراً نشطة

**الاستخدام:**
```php
Route::middleware(['auth', 'active-roles'])->group(function () {
    // Routes that require active roles
});
```

### 4. CheckPermission (alias: `permission`)

**الغرض:** التحقق من صلاحية محددة للمستخدم.

**السلوك:**
- يتحقق من امتلاك المستخدم للصلاحية المطلوبة عبر `spatie/laravel-permission`،
  مُقيَّمةً ضمن **team = tenant الحالي** (لا توجد قائمة `permission_names` في
  الجلسة — الصلاحيات تُحسَب من أدوار المستخدم تحت الـ team النشط).
- إذا لم تكن الصلاحية موجودة، يتم إرجاع خطأ 403.

**الاستخدام:**
```php
Route::middleware(['auth', 'permission:users.create'])->group(function () {
    // Routes that require 'users.create' permission
});
```

> لا يوجد alias باسم `role`. للتحقق على مستوى المستأجر تحديداً تتوفّر أيضاً
> `tenant.permission` (`EnsureTenantPermission`) و`tenant.role`
> (`EnsureTenantRole`) و`tenant.scope` (`EnsureTenantScope`).

### 5. الوسائط الإضافية المسجَّلة (aliases فعلية)

من `bootstrap/app.php` (نمط Laravel 12، لا يوجد `Http/Kernel.php`):

| alias | الصنف | الغرض |
|---|---|---|
| `context` | `EnsureContext` | وجود سياق نشط |
| `active-user` | `EnsureActiveUser` | المستخدم نشط (غير معلَّق) |
| `active-tenant-context` | `EnsureActiveTenantContext` | صحة سياق المستأجر |
| `active-roles` | `EnsureActiveRoles` | وجود أدوار نشطة |
| `permission` | `CheckPermission` | صلاحية محددة |
| `tenant.permission` | `EnsureTenantPermission` | صلاحية بنطاق المستأجر |
| `tenant.role` | `EnsureTenantRole` | دور بنطاق المستأجر |
| `tenant.scope` | `EnsureTenantScope` | فرض نطاق المستأجر |

## إدارة Teams (spatie/laravel-permission) — الآلية الفعلية

الحزمة مُفعَّل فيها `teams => true` مع `team_foreign_key => 'tenant_id'` و
`team_resolver => App\Permission\TenantTeamResolver`. **لا يوجد alias باسم
`SetPermissionsTeamId`**؛ يُدار الـ team id تلقائياً لا يدوياً عبر route:

- الوسيط العالمي `App\Http\Middleware\SyncSpatieTenantFromSessionContext`
  (مُلحَق عالمياً في `bootstrap/app.php`) يستدعي `Tenant::makeCurrent()` من
  `session('context.tenant_id')`.
- `makeCurrent()` يُطلق مهمة تعدد المستأجرين
  `App\Multitenancy\Tasks\SyncPermissionTeamTask` التي تنادي
  `setPermissionsTeamId($tenant->getKey())` (و`null` عند `forgetCurrent()`).
- `TenantTeamResolver::getPermissionsTeamId()` يحلّ: `setPermissionsTeamId()`
  الصريح ← `Tenant::current()` ← `session('context.tenant_id')`.
- الخدمات التي تكتب أدواراً (مثل `SelectRoleService`) تستدعي
  `setPermissionsTeamId()` صراحةً حول الكتابة.

## ترتيب تنفيذ Middleware

الترتيب الفعلي (مبسّط) في `bootstrap/app.php`:

1. **SetLocale** - تعيين اللغة
2. **SyncSpatieTenantFromSessionContext** (عالمي) - ضبط team id من السياق
3. **EnsureContext** - التحقق من وجود سياق
4. **EnsureActiveTenantContext** - التحقق من صحة سياق المستأجر
5. **EnsureActiveRoles** - التحقق من وجود أدوار نشطة
6. **CheckPermission** / `tenant.permission` - التحقق من صلاحيات محددة

## التكامل مع spatie/laravel-permission

### تفعيل Teams Feature

لتفعيل ميزة Teams في حزمة `spatie/laravel-permission`:

1. تحديث `config/permission.php`:
```php
'teams' => true,
```

2. تحديث Middleware لتعيين `team_id` بناءً على سياق المستأجر

3. التأكد من أن Migrations تحتوي على `team_id` في الجداول المطلوبة

### مسح ذاكرة التخزين المؤقت

عند تغيير السياق (خاصة عند تغيير المستأجر)، يجب مسح ذاكرة التخزين المؤقت:

```php
app(\Spatie\Permission\PermissionRegistrar::class)->forgetCachedPermissions();
```

يتم استدعاء هذا عادة في:
- Middleware عند تغيير المستأجر
- ContextBuilder عند بناء السياق الجديد
- SelectTenantService عند اختيار مستأجر جديد
