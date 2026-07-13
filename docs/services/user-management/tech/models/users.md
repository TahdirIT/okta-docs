# `users`

## الغرض

يمثل سجل المستخدم الأساسي (Core user record) في المنصة.

في `okta-web` لا يتم تخزين `username/email/phone/national_id` كأعمدة مباشرة في جدول `users`، بل يتم تخزينها في جدول منفصل `user_identifiers`، بينما كلمة المرور تُخزن في `user_credentials`.

## أعمدة موجودة في المشروع (`okta-web`)

- **id**: `bigint` (PK)
- **name**: `varchar` nullable
- **status**: `varchar` default `active`  (`active | suspended`)
- **default_country_id**: `bigint` nullable (FK -> `countries.id`, null on delete)
- **last_login_at**: `timestamp` nullable
- **created_at / updated_at**: timestamps
- **deleted_at**: soft deletes

## علاقات مرتبطة (في المشروع)

- **identifiers**: `hasMany(UserIdentifier)` (راجع: `user_identifiers.md`)
- **primaryIdentifier**: `hasOne(UserIdentifier)` مع `is_primary = true`
- **credential**: `hasOne(UserCredential)` (راجع: `user_credentials.md`)

## ملاحظات مرتبطة بالتفعيل/التحقق

- مفهوم "تحقق الجوال/البريد" يتم تمثيله عبر `user_identifiers.verified_at` لكل Identifier (راجع: `user_identifiers.md`).
- في سياق `tenant-registration`، تقييد الاستخدام قبل التفعيل يعتمد على إلزامية القنوات حسب إعدادات الدولة في:
  - `docs/services/countries-management/guide/entity-registration-customizations.md`

