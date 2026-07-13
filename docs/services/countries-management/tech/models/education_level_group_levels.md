# `education_level_group_levels` (Pivot مقترح)

## الغرض

ربط المستويات (`education_levels`) بمجموعاتها (`education_level_groups`) لكل دولة.

## الأعمدة (مقترح Laravel 12 + PostgreSQL)

- **education_level_group_id**: `bigint` (FK → `education_level_groups.id`)
- **education_level_id**: `bigint` (FK → `education_levels.id`)
- (اختياري) **sort_order**: `int` default `0` (للتحكم بترتيب العرض داخل المجموعة)

## الفهارس/القيود

- **primary key**: (`education_level_group_id`, `education_level_id`)
- **index**: (`education_level_id`)


