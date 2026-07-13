# `academic_years`

## الغرض

إدارة الأعوام الدراسية على مستوى الدولة.

## الأعمدة (مقترح Laravel 12 + PostgreSQL)

- **id**: `bigint` (PK)
- **ulid**: `char(26)` unique (ULID للاستخدام في APIs العامة)
- **country_id**: `bigint` (FK → `countries.id`)
- **name_ar**: `varchar` (يفضل non-null)
- **name_en**: `varchar` nullable
- **name_json**: `jsonb` nullable
- **start_date**: `timestamptz`
- **end_date**: `timestamptz`
- **is_active**: `boolean` default `false`
- **deleted_at**: `timestamptz` nullable
- **created_at / updated_at**: `timestamptz`

## العلاقات

- **belongsTo**: `countries`
- **hasMany**: `terms`

## الفهارس/القيود

- **unique**: (`ulid`)
- **index**: (`country_id`)
- (مقترح) **check**: `end_date >= start_date`
- (مقترح) منع وجود أكثر من `is_active=true` لكل دولة (عبر منطق التطبيق أو partial unique index في Postgres)


