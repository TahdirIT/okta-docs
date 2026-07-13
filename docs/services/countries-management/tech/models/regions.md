# `regions`

## الغرض

تعريف المناطق التابعة لدولة (Regions) ضمن الهيكل الجغرافي.

> **ملاحظة مهمة:** هذا الجدول يعتمد على جدول `states` من حزمة `nnjeim/world`. في البكج يُطلق عليها "states" (الولايات/المحافظات) بينما في المشروع نستخدم مصطلح "regions" (المناطق) للتوافق مع المصطلحات المحلية.

## الحزمة المستخدمة

**`nnjeim/world`** - جدول `states` في البكج يطابق جدول `regions` في المشروع.

### الأعمدة من البكج (nnjeim/world)

البكج يوفر الأعمدة التالية في جدول `states`:
- **id**: `bigint` (PK)
- **name**: `varchar` (اسم المنطقة/الولاية بالإنجليزية)
- **country_id**: `bigint` (FK → `countries.id`)
- **country_code**: `varchar(3)` (كود الدولة بمعيار ISO 3166-1 alpha-3)
- **state_code**: `varchar(5)` nullable (كود المنطقة/الولاية)
- **type**: `varchar` nullable (نوع المنطقة: state, province, region, etc.)
- **latitude**: `varchar` nullable (خط العرض)
- **longitude**: `varchar` nullable (خط الطول)

### الأعمدة الإضافية المضافة للمشروع

- **ulid**: `char(26)` unique (ULID للاستخدام في APIs العامة)
- **capital_city_id**: `bigint` nullable (مرجع منطقي لمدينة العاصمة إن لزم)
- **name_ar**: `varchar` nullable (اسم المنطقة بالعربية - إضافة محلية)
- **name_en**: `varchar` nullable (يمكن استخدامه كـ override لـ `name` من البكج)
- **name_json**: `jsonb` nullable (للدعم متعدد اللغات عند الحاجة)
- **code**: `varchar` nullable (مرادف لـ `state_code` من البكج - للتوافق مع الكود الحالي)
- **national_id**: `varchar` nullable (كما يظهر في الدليل - حقل خاص بالمشروع)
- **center**: `varchar` nullable (إحداثيات نصية أو WKT إن لزم - يمكن استخدام `latitude` و `longitude` من البكج)
- **boundaries**: `geometry` nullable (يتطلب PostGIS - حقل خاص بالمشروع)
- **population**: `bigint` nullable (حقل خاص بالمشروع)
- **deleted_at**: `timestamptz` nullable (Soft Deletes)
- **created_at / updated_at**: `timestamptz`

## العلاقات

- **belongsTo**: `countries`
- **hasMany**: `cities`
- **hasMany**: `districts` (إن كان الأحياء مرتبطة أيضاً بالمنطقة)

## الفهارس/القيود

- **unique**: (`ulid`)
- **index**: (`country_id`)
- **index**: (`country_code`) - من البكج
- (اختياري) **unique**: (`country_id`, `code`)
- (اختياري) **unique**: (`country_id`, `state_code`) - من البكج

## الاستخدام مع البكج

### العلاقة مع البكج

- جدول `regions` في المشروع = جدول `states` في البكج `nnjeim/world`
- يتم استيراد البيانات الأساسية من البكج ثم إضافة الحقول الإضافية

### الوصول للبيانات

يمكن الوصول لبيانات المناطق عبر:

1. **النموذج المخصص للمشروع** (`App\Models\Region`) - مع الحقول الإضافية
2. **نموذج البكج** (`\Nnjeim\World\Models\State`) - للوصول المباشر لبيانات البكج
3. **World Facade** - للاستعلامات والتصفية:
   ```php
   use Nnjeim\World\World;
   $action = World::states(['filters' => ['country_id' => 1]]);
   ```

### التوصيات

- استخدام `country_code` من البكج للربط مع `countries.iso3`
- استخدام `state_code` من البكج كقيمة لـ `code` عند الحاجة
- استخدام `latitude` و `longitude` من البكج كقيمة لـ `center` عند الحاجة


