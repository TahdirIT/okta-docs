# `tenant_user` (Tenant ↔ User)

## الغرض

يمثل جدول الربط بين `tenants` و `users` في `okta-web`.  
في سياق `tenant-registration` يُستخدم لربط **المستخدم المالك/المدير** بالـ Tenant الجديد، ثم يمكن إسناد أدوار ضمن هذا الـ Tenant.

## الأعمدة (مطابقة لـ `okta-web`)

- **id**: `bigint` (PK)
- **tenant_id**: `bigint` (FK → `tenants.id`) `cascadeOnDelete`
- **user_id**: `bigint` (FK → `users.id`) `cascadeOnDelete`
- **created_at / updated_at**: `timestamp`
- **deleted_at**: `timestamp` (Soft Deletes)

## القيود/الفهارس (مطابقة لـ `okta-web`)

- **unique**: (`tenant_id`, `user_id`)

## العلاقات (على مستوى Eloquent)

- `TenantUser`:
  - **belongsTo**: `tenant`
  - **belongsTo**: `user`
  - **roles()**: تُحلّ عبر جدول Spatie القياسي `model_has_roles` مُفلتَراً بـ
    `tenant_id` + `model_type = User` (ميزة Teams) — **لا** يوجد جدول ربط خاص.

- `Tenant`:
  - **hasMany**: `userLinks`
  - **belongsToMany**: `users` عبر `tenant_user` مع تجاهل روابط الـ pivot المحذوفة (`deleted_at`)

- `User`:
  - **hasMany**: `tenantLinks`
  - **belongsToMany**: `tenants` عبر `tenant_user` مع تجاهل روابط الـ pivot المحذوفة (`deleted_at`)

## ملاحظات

- جدول `tenant_user` يحمل عمودَي `tenant_id` و`user_id` فقط (+ الطوابع
  و`deleted_at`) — **بلا** عمود `role` أو `is_owner`. إسناد الأدوار داخل الجهة
  يتم عبر Spatie (`model_has_roles` مع `tenant_id`)، لا عبر أعمدة على هذا الجدول.
- أثناء التسجيل، `RegisterTenant` ينشئ صفّ `tenant_user` ثم يُسند دور المالك
  `tenant-admin` (الثابت `TENANT_OWNER_ROLE`) ضمن سياق team = الجهة الجديدة.

