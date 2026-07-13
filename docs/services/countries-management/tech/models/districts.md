# `districts`

## الغرض

تعريف الأحياء التابعة لمدينة (وممكن ربطها مباشرة بالمنطقة أيضاً حسب البيانات المستوردة).

> **ملاحظة مهمة:** هذا الجدول **خاص بالمشروع** ولا يوجد في حزمة `nnjeim/world`. البكج يوفر فقط: الدول (countries)، الولايات/المناطق (states)، والمدن (cities). الأحياء (districts) هي مستوى إضافي خاص بالمشروع لإدارة التقسيمات الجغرافية الأصغر.

## الحزمة المستخدمة

**`nnjeim/world`** - لا يوفر هذا الجدول. الأحياء (districts) هي إضافة محلية للمشروع.

## الأعمدة (مقترح Laravel 12 + PostgreSQL)

- **id**: `bigint` (PK)
- **ulid**: `char(26)` unique (ULID للاستخدام في APIs العامة)
- **city_id**: `bigint` (FK → `cities.id`)
- **region_id**: `bigint` (FK → `regions.id`)
- **name_ar**: `varchar` nullable
- **name_en**: `varchar` nullable
- **name_json**: `jsonb` nullable
- **deleted_at**: `timestamptz` nullable
- **created_at / updated_at**: `timestamptz`

## العلاقات

- **belongsTo**: `cities`
- **belongsTo**: `regions`

## الفهارس/القيود

- **unique**: (`ulid`)
- **index**: (`city_id`)
- **index**: (`region_id`)

## ملاحظات حول البكج

- حزمة `nnjeim/world` **لا توفر** بيانات الأحياء (districts)
- هذا الجدول خاص بالمشروع ويتم إدارته محلياً
- يمكن ربط الأحياء بالمدن (`cities`) التي تأتي من البكج
- يمكن أيضاً ربطها مباشرة بالمناطق (`regions`) التي تطابق `states` من البكج


