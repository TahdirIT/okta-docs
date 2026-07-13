# `holiday_schedules`

## الغرض

جدول الإجازات والأنشطة على مستوى الدولة/المدرسة.

## الأعمدة (مقترح Laravel 12 + PostgreSQL)

- **id**: `bigint` (PK)
- **ulid**: `char(26)` unique (ULID للاستخدام في APIs العامة)
- **title**: `jsonb` (يدعم العربية/الإنجليزية)
- **related_type**: `varchar` (polymorphic: `App\\Models\\Country` أو `App\\Models\\School`)
- **related_id**: `bigint`
- **type**: `varchar` (مثل: `HOLIDAY`, `ACTIVITY`)
- **start_at**: `date`
- **end_at**: `date`
- **register_absent**: `boolean` default `false`
- **deleted_at**: `timestamptz` nullable
- **created_at / updated_at**: `timestamptz`

## العلاقات

- **morphTo**: `related`

## الفهارس/القيود

- **unique**: (`ulid`)
- **index**: (`related_type`, `related_id`)
- **index**: (`start_at`, `end_at`)
- (مقترح) **check**: `end_at >= start_at`
- (مقترح Postgres) منع تداخل الإجازات ضمن نفس النطاق يمكن تنفيذه عبر:
  - منطق تطبيق + استعلامات overlap
  - أو `EXCLUDE USING gist` على `daterange(start_at,end_at,'[]')` مع `related_*` (يتطلب تصميم إضافي)


