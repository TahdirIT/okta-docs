# Tenant Scoping للموديلات

## المشكلة

في بيئة متعددة المستأجرين، الموديلات التي تحتوي على `tenant_id` (مثل الطلاب، الفصول، المعلمين...) يجب أن تُفلتر تلقائياً بحيث لا يرى أي مستأجر بيانات مستأجر آخر.

## الحل: Trait `BelongsToTenant`

**المسار:** `app/Models/Concerns/BelongsToTenant.php`

يضيف هذا الـ trait:
1. **Global Scope** يفلتر `WHERE tenant_id = Tenant::current()->id` تلقائياً على كل استعلام.
2. **Auto-fill** يُعبّئ `tenant_id` تلقائياً عند إنشاء سجل جديد.
3. **`tenant()` relationship** ربط `belongsTo(Tenant::class)`.
4. **`withoutTenantScope()`** للاستعلام بدون فلتر عند الحاجة.

## الاستخدام

```php
use App\Models\Concerns\BelongsToTenant;

class Student extends Model
{
    use BelongsToTenant;

    protected $fillable = ['tenant_id', 'name', ...];
}
```

### أمثلة

```php
// يُرجع طلاب المستأجر الحالي فقط (Tenant::current())
Student::all();

// إنشاء طالب — tenant_id يُعبَّأ تلقائياً
Student::create(['name' => 'أحمد']);

// استعلام بدون فلتر المستأجر (للنظام فقط)
Student::withoutTenantScope()->where('name', 'like', '%أحمد%')->get();
```

### Blade Directives

```blade
@can('users.view')
    {{-- يعمل تلقائياً في سياق المستأجر الحالي --}}
@endcan
```

## السلوك بحسب السياق

| السياق | السلوك |
|---|---|
| مستأجر نشط | `WHERE tenant_id = {id}` مضاف تلقائياً |
| سياق النظام (بلا مستأجر) | لا يُضاف أي فلتر |
| `withoutTenantScope()` | لا يُضاف أي فلتر بغض النظر عن السياق |

## الموديلات التي تستخدم هذا الـ Trait

| الموديل | ملاحظة |
|---|---|
| `TenantLoginMethod` | طرق تسجيل الدخول الخاصة بكل مستأجر |

> كل موديل يُضاف للنظام ويحتوي على `tenant_id` يجب أن يستخدم هذا الـ trait.

## الموديلات التي لا تستخدمه

| الموديل | السبب |
|---|---|
| `TenantUser` | جدول الربط بين User و Tenant — لا يُفلتر لأنه يُستخدم في بناء السياق |
| `Role` | القوالب مشتركة (`tenant_id = null`)؛ التعيين عبر `model_has_roles` |
| `Permission` | مشتركة بين جميع المستأجرين |
| `User` | المستخدم قد ينتمي لأكثر من مستأجر |

## متطلبات إضافة موديل جديد

1. أضف عمود `tenant_id` في migration:
   ```php
   $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
   ```
2. أضف `tenant_id` في `$fillable`.
3. أضف `use BelongsToTenant;` في الموديل.
4. **لا تضف** علاقة `tenant()` يدوياً، يوفرها الـ trait.
