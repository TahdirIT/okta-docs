# `subject_education_levels` (Pivot مقترح)

## الغرض

ربط المواد (`subjects`) بالمستويات/الصفوف (`education_levels`) على مستوى الدولة.

## الأعمدة (مقترح Laravel 12 + PostgreSQL)

- **subject_id**: `bigint` (FK → `subjects.id`)
- **education_level_id**: `bigint` (FK → `education_levels.id`)
- (اختياري) **sort_order**: `int` default `0`

## الفهارس/القيود

- **primary key**: (`subject_id`, `education_level_id`)
- **index**: (`education_level_id`)


