# `countries`

## الغرض

تعريف الدول كمرجع أساسي تُبنى عليه إعدادات الدولة وبقية البيانات التابعة لها.

> **ملاحظة مهمة:** هذا الجدول يعتمد على حزمة `nnjeim/world` التي توفر البيانات الأساسية للدول. يتم استيراد البيانات من البكج ثم إضافة الحقول الإضافية المطلوبة للمشروع.

## الحزمة المستخدمة

**`nnjeim/world`** - حزمة Laravel توفر قائمة شاملة بالدول والولايات والمدن والعملات والمناطق الزمنية واللغات.

### الأعمدة من البكج (nnjeim/world)

البكج يوفر الأعمدة التالية في جدول `countries`:
- **id**: `bigint` (PK)
- **name**: `varchar` (اسم الدولة بالإنجليزية)
- **iso2**: `varchar(2)` (كود الدولة بمعيار ISO 3166-1 alpha-2، مثل: SA, KW)
- **iso3**: `varchar(3)` (كود الدولة بمعيار ISO 3166-1 alpha-3)
- **phone_code**: `varchar(5)` (رمز الهاتف الدولي)
- **native**: `varchar` nullable (اسم الدولة باللغة الأصلية)
- **region**: `varchar` nullable (المنطقة الجغرافية)
- **subregion**: `varchar` nullable (المنطقة الفرعية)
- **latitude**: `varchar` nullable (خط العرض)
- **longitude**: `varchar` nullable (خط الطول)
- **emoji**: `varchar` nullable (رمز العلم كإيموجي)
- **emojiU**: `varchar` nullable (رمز العلم بصيغة Unicode)

### الأعمدة الإضافية المضافة للمشروع

- **ulid**: `char(26)` unique (ULID للاستخدام في APIs العامة)
- **name_ar**: `varchar` nullable (اسم الدولة بالعربية - إضافة محلية)
- **name_en**: `varchar` nullable (يمكن استخدامه كـ override لـ `name` من البكج)
- **name_json**: `jsonb` nullable (للدعم متعدد اللغات عند الحاجة)
- **code**: `varchar` nullable (مرادف لـ `iso2` من البكج - للتوافق مع الكود الحالي)
- **flag**: `varchar` nullable (يمكن استخدام `emoji` من البكج)
- **timezone**: `varchar` nullable (مثل: `Asia/Riyadh` - يمكن ربطه بجدول `timezones` من البكج)
- **active**: `boolean` default `true` (حقل خاص بالمشروع لتفعيل/تعطيل الدولة)
- **deleted_at**: `timestamptz` nullable (Soft Deletes)
- **created_at / updated_at**: `timestamptz`

## العلاقات

- **hasMany**: `regions`
- **hasMany**: `academic_years`
- **hasMany**: `education_level_groups`
- (اختياري) **hasMany**: `education_levels`
- (اختياري) **hasMany**: `subjects`
- **morphMany**: `holiday_schedules` (via `related_type/related_id`)
- **morphOne**: `model_settings` (via `model_type/model_id`)

## الفهارس/القيود

- **unique**: (`iso2`) - من البكج
- **unique**: (`code`) - إن كانت مطلوبة كقيمة فريدة (مرادف لـ `iso2`)
- **unique**: (`ulid`)
- **index**: (`active`)
- **index**: (`iso2`) - من البكج

## الاستخدام مع البكج

### استيراد البيانات

يتم استيراد بيانات الدول من البكج `nnjeim/world` عبر:

```php
use Nnjeim\World\Actions\SeedAction;
// أو
php artisan world:seed
```

### الوصول للبيانات

يمكن الوصول لبيانات الدول عبر:

1. **النموذج المخصص للمشروع** (`App\Models\Country`) - مع الحقول الإضافية
2. **نموذج البكج** (`\Nnjeim\World\Models\Country`) - للوصول المباشر لبيانات البكج
3. **World Facade** - للاستعلامات والتصفية:
   ```php
   use Nnjeim\World\World;
   $action = World::countries();
   ```

### التوصيات

- استخدام `iso2` من البكج كمعرف فريد للدولة
- ربط `timezone` بجدول `timezones` من البكج عند الحاجة
- استخدام `emoji` من البكج كقيمة لـ `flag` عند الحاجة


