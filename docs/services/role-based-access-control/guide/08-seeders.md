# البذور (Seeders)

## نظرة عامة

يتم استخدام Seeders لإنشاء البيانات الأولية للأدوار والصلاحيات في النظام. تضمن Seeders وجود الأدوار الثابتة والصلاحيات الأساسية المطلوبة لعمل النظام بشكل صحيح.

## الأدوار الثابتة المطلوبة

### على مستوى النظام (System Scope)

#### 1. superadmin

**الخصائص:**
- الاسم: `superadmin`
- النطاق: `system`
- المستأجر: `null`
- العنوان بالعربية: `مدير النظام`
- العنوان بالإنجليزية: `Super Administrator`

**الصلاحيات المقترحة:**
- جميع الصلاحيات على مستوى النظام
- إدارة الأدوار والصلاحيات
- إدارة المستخدمين
- إدارة المستأجرين
- الوصول لجميع الإعدادات

### على مستوى المستأجر (Tenant Scope)

#### 1. admin

**الخصائص:**
- الاسم: `admin`
- النطاق: `tenant`
- المستأجر: مرتبط بمستأجر محدد
- العنوان بالعربية: `مدير المستأجر`
- العنوان بالإنجليزية: `Tenant Administrator`

**الصلاحيات المقترحة:**
- جميع الصلاحيات على مستوى المستأجر
- إدارة الأدوار والصلاحيات على مستوى المستأجر
- إدارة المستخدمين في المستأجر
- إدارة إعدادات المستأجر

#### 2. teacher

**الخصائص:**
- الاسم: `teacher`
- النطاق: `tenant`
- المستأجر: مرتبط بمستأجر محدد
- العنوان بالعربية: `معلم`
- العنوان بالإنجليزية: `Teacher`

**الصلاحيات المقترحة:**
- عرض المواد الدراسية المرتبطة به
- عرض الطلاب المرتبطين به
- إدارة الدرجات والتقييمات
- إرسال الإشعارات للطلاب

#### 3. student

**الخصائص:**
- الاسم: `student`
- النطاق: `tenant`
- المستأجر: مرتبط بمستأجر محدد
- العنوان بالعربية: `طالب`
- العنوان بالإنجليزية: `Student`

**الصلاحيات المقترحة:**
- عرض المواد الدراسية المسجلة
- عرض الدرجات والنتائج
- عرض الجدول الدراسي
- الوصول للمواد التعليمية

## هيكل Seeder

### RoleSeeder

```php
<?php

namespace Database\Seeders;

use App\Models\Role;
use Illuminate\Database\Seeder;

class RoleSeeder extends Seeder
{
    public function run(): void
    {
        // System roles
        $this->createSystemRoles();
        
        // Note: Tenant roles should be created per tenant
        // This can be done in TenantSeeder or when creating a new tenant
    }
    
    private function createSystemRoles(): void
    {
        Role::firstOrCreate(
            ['name' => 'superadmin', 'guard_name' => 'web', 'scope' => 'system'],
            [
                'title_ar' => 'مدير النظام',
                'title_en' => 'Super Administrator',
                'tenant_id' => null,
            ]
        );
    }
}
```

### TenantRoleSeeder

```php
<?php

namespace Database\Seeders;

use App\Models\Role;
use App\Models\Tenant;
use Illuminate\Database\Seeder;

class TenantRoleSeeder extends Seeder
{
    public function run(): void
    {
        // Create tenant roles for all existing tenants
        Tenant::each(function ($tenant) {
            $this->createTenantRoles($tenant);
        });
    }
    
    public function createTenantRoles(Tenant $tenant): void
    {
        // Admin role
        Role::firstOrCreate(
            [
                'name' => 'admin',
                'guard_name' => 'web',
                'scope' => 'tenant',
                'tenant_id' => $tenant->id,
            ],
            [
                'title_ar' => 'مدير المستأجر',
                'title_en' => 'Tenant Administrator',
            ]
        );
        
        // Teacher role
        Role::firstOrCreate(
            [
                'name' => 'teacher',
                'guard_name' => 'web',
                'scope' => 'tenant',
                'tenant_id' => $tenant->id,
            ],
            [
                'title_ar' => 'معلم',
                'title_en' => 'Teacher',
            ]
        );
        
        // Student role
        Role::firstOrCreate(
            [
                'name' => 'student',
                'guard_name' => 'web',
                'scope' => 'tenant',
                'tenant_id' => $tenant->id,
            ],
            [
                'title_ar' => 'طالب',
                'title_en' => 'Student',
            ]
        );
    }
}
```

## إنشاء الأدوار عند إنشاء مستأجر جديد

عند إنشاء مستأجر جديد، يجب إنشاء الأدوار الثابتة لهذا المستأجر تلقائياً. يمكن القيام بذلك في:

### 1. Tenant Model Observer

```php
<?php

namespace App\Observers;

use App\Models\Tenant;
use App\Models\Role;

class TenantObserver
{
    public function created(Tenant $tenant): void
    {
        $this->createDefaultRoles($tenant);
    }
    
    private function createDefaultRoles(Tenant $tenant): void
    {
        $roles = [
            [
                'name' => 'admin',
                'title_ar' => 'مدير المستأجر',
                'title_en' => 'Tenant Administrator',
            ],
            [
                'name' => 'teacher',
                'title_ar' => 'معلم',
                'title_en' => 'Teacher',
            ],
            [
                'name' => 'student',
                'title_ar' => 'طالب',
                'title_en' => 'Student',
            ],
        ];
        
        foreach ($roles as $roleData) {
            Role::create([
                'name' => $roleData['name'],
                'guard_name' => 'web',
                'scope' => 'tenant',
                'tenant_id' => $tenant->id,
                'title_ar' => $roleData['title_ar'],
                'title_en' => $roleData['title_en'],
            ]);
        }
    }
}
```

### 2. في Service إنشاء المستأجر

```php
public function createTenant(array $data): Tenant
{
    $tenant = Tenant::create($data);
    
    // Create default roles
    app(TenantRoleSeeder::class)->createTenantRoles($tenant);
    
    return $tenant;
}
```

## ربط الصلاحيات بالأدوار

يمكن ربط الصلاحيات بالأدوار في Seeder منفصل أو في نفس Seeder:

```php
public function assignPermissionsToRoles(): void
{
    $superadmin = Role::where('name', 'superadmin')->where('scope', 'system')->first();
    
    if ($superadmin) {
        // Get all system permissions
        $systemPermissions = Permission::where('scope', 'system')->pluck('id');
        $superadmin->permissions()->sync($systemPermissions);
    }
    
    // Similar for tenant roles
    Tenant::each(function ($tenant) {
        $admin = Role::where('name', 'admin')
            ->where('scope', 'tenant')
            ->where('tenant_id', $tenant->id)
            ->first();
            
        if ($admin) {
            // Get all tenant permissions for this tenant
            $tenantPermissions = Permission::where('scope', 'tenant')
                ->where('tenant_id', $tenant->id)
                ->pluck('id');
            $admin->permissions()->sync($tenantPermissions);
        }
    });
}
```

## ملاحظات مهمة

1. **الأدوار الثابتة**: الأدوار الثابتة (`superadmin`, `finance-admin`, `tenant-admin`) يجب أن تكون موجودة دائماً ولا يمكن حذفها.

2. **الأدوار لكل مستأجر**: الأدوار على مستوى المستأجر يجب إنشاؤها لكل مستأجر بشكل منفصل.

3. **العناوين**: يجب إضافة العناوين بالعربية والإنجليزية للأدوار الثابتة لتحسين تجربة المستخدم.

4. **الصلاحيات**: يمكن ربط الصلاحيات بالأدوار في Seeder منفصل أو عند إنشاء الأدوار.

5. **التحقق من الوجود**: استخدم `firstOrCreate` بدلاً من `create` لتجنب الأخطاء عند تشغيل Seeder عدة مرات.
