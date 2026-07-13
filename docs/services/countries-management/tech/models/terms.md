# `terms`

## الغرض

الفصول الدراسية التابعة لعام دراسي (مثل: الفصل الأول/الثاني أو ثلاثيات).

## الأعمدة (مقترح Laravel 12 + PostgreSQL)

- **id**: `bigint` (PK)
- **ulid**: `char(26)` unique (ULID للاستخدام في APIs العامة)
- **academic_year_id**: `bigint` (FK → `academic_years.id`)
- **name_ar**: `varchar`
- **name_en**: `varchar` nullable
- **name_json**: `jsonb` nullable
- **start_date**: `timestamptz`
- **end_date**: `timestamptz`
- **is_active**: `boolean` default `true`
- **deleted_at**: `timestamptz` nullable
- **created_at / updated_at**: `timestamptz`

## العلاقات

- **belongsTo**: `academic_years`

## الفهارس/القيود

- **unique**: (`ulid`)
- **index**: (`academic_year_id`)
- (مقترح) **check**: `end_date >= start_date`

## ملاحظات

- في الدليل يوجد إعداد على مستوى الدولة: `active_term_id` (يفضل تخزينه في `model_settings` أو جدول إعدادات صريح).


