# الصلاحيات (Permissions)

## أدوار معنية
- مسؤول المنصة: إدارة تعريفات الإشعارات، القوالب الافتراضية، وسياسات اللغة حسب الدولة.
- مسؤول الجهة: تخصيص القوالب واللغة الافتراضية وتفضيلات القنوات للجهة.
- المستخدم النهائي: إدارة تفضيلات الاشتراك/الإلغاء إن كانت مُفعّلة.

## توزيع الصلاحيات
- قراءة/قائمة التعريفات: متاحة للأدوار الإدارية بحسب السياق.
- إنشاء/تعديل/حذف تعريفات: حصر على مستوى المنصة.
- تخصيص قوالب الجهة: حصر على مستوى الجهة.
- إدارة التفضيلات: حسب تعريف الإشعار وسياسة الجهة.

## أكواد الصلاحيات (وفق معيار التسمية)

تُفرَض عبر `authorize()` في مكوّنات الإعدادات (مثل
`notifications_management.definitions.delete`):

- notifications_management.definitions.view
- notifications_management.definitions.create
- notifications_management.definitions.update
- notifications_management.definitions.delete
- notifications_management.templates.view
- notifications_management.templates.create
- notifications_management.templates.update
- notifications_management.templates.delete
- notifications_management.locale_policies.update_country_default
- notifications_management.tenant.overrides.update_default_locale
- notifications_management.tenant.templates.override
- notifications_management.preferences.view
- notifications_management.preferences.update

ملاحظات:
- feature = notifications_management
- resources = definitions | templates | locale_policies | tenant.* | preferences
- actions = view|create|update|delete … (انظر `docs/tech-standards/permissions-naming.md`)

## أمثلة تكامل الكود

### 1) ربط الصلاحية على مستوى Route
```php
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Admin\NotificationDefinitionsController;

Route::middleware(['auth', 'can:notifications_management.definitions.view'])
    ->get('/admin/notification-definitions', [NotificationDefinitionsController::class, 'index']);

Route::middleware(['auth', 'can:notifications_management.definitions.create'])
    ->post('/admin/notification-definitions', [NotificationDefinitionsController::class, 'store']);
```

### 2) تفويض داخل الـ Controller
```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;

public function store(Request $request)
{
    Gate::authorize('notifications_management.definitions.create');

    // تنفيذ الإنشاء...
}
```

### 3) استخدامه في Blade
```php
@can('notifications_management.templates.update')
    <button type="button" class="btn btn-primary">تعديل القالب</button>
@endcan
```

### 4) ردّ API عند عدم التوافر
```json
{
  "message": "This action is unauthorized."
}
```

### 5) تسجيل/زرع (Seeding) الصلاحيات (مثال شكلي)
```php
$permissions = [
    'notifications_management.definitions.view',
    'notifications_management.definitions.create',
    'notifications_management.definitions.update',
    'notifications_management.definitions.delete',
    'notifications_management.templates.view',
    'notifications_management.templates.create',
    'notifications_management.templates.update',
    'notifications_management.templates.delete',
    'notifications_management.locale_policies.update_country_default',
    'notifications_management.tenant.overrides.update_default_locale',
    'notifications_management.tenant.templates.override',
    'notifications_management.preferences.view',
    'notifications_management.preferences.update',
];

// إنشاء الصلاحيات وإسنادها للأدوار المناسبة
```

