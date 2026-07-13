# `cities`

## الغرض

تعريف المدن التابعة لمنطقة.

> **ملاحظة مهمة:** هذا الجدول يعتمد على حزمة `nnjeim/world` التي توفر البيانات الأساسية للمدن. يتم استيراد البيانات من البكج ثم إضافة الحقول الإضافية المطلوبة للمشروع.

## الحزمة المستخدمة

**`nnjeim/world`** - حزمة Laravel توفر قائمة شاملة بالمدن مرتبطة بالدول والولايات.

### الأعمدة من البكج (nnjeim/world)

البكج يوفر الأعمدة التالية في جدول `cities`:
- **id**: `bigint` (PK)
- **name**: `varchar` (اسم المدينة بالإنجليزية)
- **country_id**: `bigint` (FK → `countries.id`)
- **country_code**: `varchar(3)` (كود الدولة بمعيار ISO 3166-1 alpha-3)
- **state_id**: `bigint` nullable (FK → `states.id` - يطابق `regions.id` في المشروع)
- **state_code**: `varchar(5)` nullable (كود المنطقة/الولاية)
- **latitude**: `varchar` nullable (خط العرض)
- **longitude**: `varchar` nullable (خط الطول)

### الأعمدة الإضافية المضافة للمشروع

- **ulid**: `char(26)` unique (ULID للاستخدام في APIs العامة)
- **region_id**: `bigint` (FK → `regions.id`) - يطابق `state_id` من البكج
- **name_ar**: `varchar` nullable (اسم المدينة بالعربية - إضافة محلية)
- **name_en**: `varchar` nullable (يمكن استخدامه كـ override لـ `name` من البكج)
- **name_json**: `jsonb` nullable (للدعم متعدد اللغات عند الحاجة)
- **deleted_at**: `timestamptz` nullable (Soft Deletes)
- **created_at / updated_at**: `timestamptz`

## العلاقات

- **belongsTo**: `regions`
- **hasMany**: `districts`

## الفهارس/القيود

- **unique**: (`ulid`)
- **index**: (`region_id`)
- **index**: (`state_id`) - من البكج
- **index**: (`country_id`) - من البكج
- **index**: (`country_code`) - من البكج

## الاستخدام مع البكج

### العلاقة مع البكج

- جدول `cities` في المشروع يعتمد على جدول `cities` في البكج `nnjeim/world`
- يتم استيراد البيانات الأساسية من البكج ثم إضافة الحقول الإضافية
- `region_id` في المشروع يطابق `state_id` في البكج

### الوصول للبيانات

يمكن الوصول لبيانات المدن عبر:

1. **النموذج المخصص للمشروع** (`App\Models\City`) - مع الحقول الإضافية
2. **نموذج البكج** (`\Nnjeim\World\Models\City`) - للوصول المباشر لبيانات البكج
3. **World Facade** - للاستعلامات والتصفية:
   ```php
   use Nnjeim\World\World;
   $action = World::cities(['filters' => ['country_id' => 1, 'state_id' => 5]]);
   ```

### التوصيات

- استخدام `state_id` من البكج كقيمة لـ `region_id` عند الاستيراد
- استخدام `latitude` و `longitude` من البكج عند الحاجة للإحداثيات


