# `education_level_groups`

## الغرض

مجموعات المراحل/المستويات على مستوى الدولة (مثل: ابتدائي، متوسط، ثانوي) لاستخدامها في تنظيم الهيكل الدراسي وربط المواد.

## الأعمدة (مقترح Laravel 12 + PostgreSQL)

- **id**: `bigint` (PK)
- **ulid**: `char(26)` unique (ULID للاستخدام في APIs العامة)
- **country_id**: `bigint` (FK → `countries.id`)
- **name_ar**: `varchar`
- **name_en**: `varchar`
- **name_json**: `jsonb` nullable
- **deleted_at**: `timestamptz` nullable
- **created_at / updated_at**: `timestamptz`

## العلاقات

- **belongsTo**: `countries`
- **belongsToMany**: `education_levels` (عبر pivot `education_level_group_levels`)

## الفهارس/القيود

- **unique**: (`ulid`)
- **index**: (`country_id`)


