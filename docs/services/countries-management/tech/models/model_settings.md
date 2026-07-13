# `model_settings`

## الغرض

تخزين إعدادات مرنة (key/value) مرتبطة بأي Model (polymorphic).

في سياق **إدارة الدول** تُستخدم عادةً لتخزين:

- **`weekdays`**: إعدادات أيام الأسبوع على مستوى الدولة
- **`active_term_id`**: الفصل النشط

## الأعمدة (مقترح Laravel 12 + PostgreSQL)

- **id**: `bigint` (PK)
- **ulid**: `char(26)` unique (ULID للاستخدام في APIs العامة)
- **model_type**: `varchar`
- **model_id**: `bigint`
- **settings**: `jsonb`
- **created_at / updated_at**: `timestamptz`

## الفهارس/القيود

- **unique**: (`model_type`, `model_id`)
- **unique**: (`ulid`)
- (اختياري) **GIN index** على `settings` لدعم queries على `settings->...`


