# `education_levels` (مقترح)

## الغرض

تمثيل المستويات/الصفوف الدراسية المرجعية على مستوى الدولة (مثل: KG-1, Grade 1…)، بدل ربطها مباشرةً بالمدرسة.

> هذا يعالج الفجوة بين الدليل (يتحدث عن تعريف المراحل لكل دولة) وبين الربط المباشر بالمراحل على مستوى المدرسة.

## الأعمدة (مقترح Laravel 12 + PostgreSQL)

- **id**: `bigint` (PK)
- **ulid**: `char(26)` unique (ULID للاستخدام في APIs العامة)
- **country_id**: `bigint` (FK → `countries.id`)
- **code**: `varchar` nullable (مثل: `KG-1`, `G1`)
- **name**: `jsonb` (مقترح بدل `name_ar/name_en` — أو يمكن الإبقاء عليهم)
- **short_name**: `jsonb` nullable
- **sort_order**: `int` default `0`
- **min_age**: `int` nullable
- **max_age**: `int` nullable
- **active**: `boolean` default `true`
- **deleted_at**: `timestamptz` nullable
- **created_at / updated_at**: `timestamptz`

## العلاقات

- **belongsTo**: `countries`
- **belongsToMany**: `education_level_groups` (عبر pivot)

## الفهارس/القيود

- **unique**: (`ulid`)
- **index**: (`country_id`)
- (اختياري) **unique**: (`country_id`, `code`)
- (اختياري) **check**: `min_age <= max_age`


