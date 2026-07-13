# `tenants`

## الغرض

تمثيل الكيان (Tenant) في مشروع `okta-web`، وهو الكيان الذي يتم:

- ربط المستخدمين به (عبر `tenant_user`)
- إعداد طرق تسجيل الدخول الخاصة به (عبر `tenant_login_methods`)

## الأعمدة (مطابقة لـ `okta-web`، عبر عدّة migrations)

- **id**: `bigint` (PK)
- **ulid**: `char(26)` unique — معرّف مبهم يُعرَّض لجسر الشركاء.
- **name**: `varchar`
- **slug**: `varchar(63)` unique (NOT NULL).
- **type**: `varchar` — نوع الجهة: `school | kindergarten | institute | academy | college | university | …` (راجع `EntityType`).
- **country_id**: `bigint` nullable (FK → `countries.id`, `nullOnDelete`).
- **education_level_group_id**: `bigint` nullable (FK → `education_level_groups`) — لقطة التسجيل.
- **registration_custom_fields**: `json` nullable — قيم الحقول الديناميكية المُلتقطة أثناء التسجيل (cast `array`).
- **status**: `varchar` default `active` — `active | suspended`.
- **parent_tenant_id**: `bigint` nullable (self-FK) — للجهات الحاوية/التابعة.
- **is_sandbox**: `boolean` default `false` (مفهرس).
- **owner_partner_tenant_id**: `bigint` nullable — الجهة الشريكة المالكة (إن وُجدت).
- **logo / phone / email / website / address**: حقول تعريف اختيارية.
- **created_at / updated_at**: `timestamp`
- **deleted_at**: `timestamp` (Soft Deletes)

## العلاقات

- **belongsTo**: `country` (→ `countries`)، `educationLevelGroup`، `parent` (self).
- **hasMany**: `userLinks` (→ `tenant_user` rows via `TenantUser`)، `children` (self).
- **belongsToMany**: `users` عبر pivot جدول `tenant_user` (مع `deleted_at` على الـ pivot).
- **hasMany**: `loginMethods` (→ `tenant_login_methods`) مع فلترة `enabled=true` و `deleted_at is null`.

## ملاحظات

- **لا يوجد** عمود `created_by_user_id` ولا `subdomain` على `tenants` (النطاق الفرعي
  قيمة من `tenant_domains.type`، لا عمود هنا). كذلك **لا يوجد** جدول
  `tenant_registration_sessions` — راجع
  [`tenant_registration_sessions.md`](./tenant_registration_sessions.md)
  (اقتراح غير مُنفَّذ؛ المعالج يحفظ حالته في جلسة Laravel).
- الحقول الديناميكية حسب الدولة/النوع **مُنفَّذة فعلاً** عبر
  `registration_custom_fields` + `education_level_group_id` (الخطوة 3 من المعالج).

