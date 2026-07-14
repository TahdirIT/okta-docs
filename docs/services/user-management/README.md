# إدارة المستخدمين (User Management)

هذا المجلد يحتوي على توثيق خدمة **إدارة المستخدمين** في المنصة.

تشمل الخدمة:

- تعريف نموذج المستخدم (User) وبياناته الأساسية.
- قواعد منع التكرار (Uniqueness) لحقول الهوية/اسم المستخدم/الجوال/البريد.
- حالات التحقق (Mobile/Email verification) وعلاقتها بتفعيل الحساب.
- نقاط التكامل مع خدمات أخرى مثل:
  - `tenant-registration` لإنشاء حساب مالك (Owner) أثناء تسجيل كيان جديد.
  - `role-based-access-control` لإدارة الأدوار والصلاحيات للمستخدمين.

## الملفات

- [نظرة عامة](./guide/01-overview.md)
- [تسجيل مستخدم (Registration)](./guide/02-user-registration.md)
- [تسجيل الدخول (Login)](./guide/03-login.md)
- [تحديث بيانات المستخدم (User Update)](./guide/04-user-update.md)
- [Tech](./tech/README.md)
- [نماذج البيانات (Models)](./tech/models/README.md)
- [وظائف/Use-cases الخدمة](./tech/service-functions.md)

## أدوات platform-admin: الحسابات المتشابهة + الدمج + الحذف

أدوات للتعامل مع الحسابات المكررة وحذف الحسابات/العملاء (نطاق system):

- **الحسابات المتشابهة**: `GET /settings/users/duplicates` (`DuplicateUsersPage`) —
  يجمع المستخدمين المتشاركين `national_id`/`phone`/`email` أو الاسم بعد التطبيع
  العربي عبر `App\Services\UserManagement\Users\FindSimilarUsers`.
- **دمج الحسابات**: modal `settings.user-merge-modal` →
  `UserManagement\Users\MergeUsers` — ينقل identifiers/credentials/روابط
  `tenant_user`/العضويات/الأدوار والسجلات المساندة إلى الحساب الأساسي، ويحذف
  المصدر soft-delete. ممنوع دمج superadmin أو الحساب الحالي.
- **حذف مستخدم**: `settings.user-delete-modal` → `UserManagement\Users\DeleteUser`
  (soft delete + إتلاف الجلسات؛ الأدوار وسجل النشاط يبقيان للتدقيق).
- **حذف عميل (Tenant)**: `settings.tenant-delete-modal` (يتطلب كتابة الاسم) →
  `Tenants\Tenants\DeleteTenant` (يمنع حذف جهة باشتراك `active`/`grace_period`).
- **الصلاحيات**: `rbac.users.merge` + `rbac.users.delete` + `tenants.delete`.
- **اختبارات**: `tests/Feature/UserManagement/{MergeUsers,FindSimilarUsers,DeleteUser}Test`،
  `tests/Feature/Settings/DeleteTenantTest`.

