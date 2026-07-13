# تحديث بيانات المستخدم (User Update)

هذا المستند يوضح كيف يتم **تحديث بيانات المستخدم** ضمن خدمة `user-management` بما يتوافق مع تصميم النماذج في `okta-web`.

## الفكرة الأساسية (حسب `okta-web`)

تحديث بيانات المستخدم لا يتم في مكان واحد فقط، بل عبر 3 موارد:

- `users`: بيانات أساسية (مثل `name/status/default_country_id/last_login_at`)
- `user_identifiers`: معرفات الدخول وقنوات التواصل (email/phone/username/national_id/SSO) + `verified_at` + `is_primary`
- `user_credentials`: كلمة المرور (`password_hash`)

مراجع النماذج:

- `docs/services/user-management/tech/models/users.md`
- `docs/services/user-management/tech/models/user_identifiers.md`
- `docs/services/user-management/tech/models/user_credentials.md`

## حالات شائعة

### 1) تحديث الاسم أو الدولة الافتراضية

- Use-cases:
  - `app/Services/UserManagement/Users/UpdateUserProfile.php`
  - `app/Services/UserManagement/Users/UpdateUserDefaultCountry.php`

### 2) تعليق/تفعيل مستخدم

- Use-case:
  - `app/Services/UserManagement/Users/UpdateUserStatus.php`

> ملاحظة: ربط الصلاحيات/منع الوصول بناء على `status` قد يتطلب Middleware/Policy حسب تصميم النظام.

### 3) تحديث البريد/الجوال/الهوية/اسم المستخدم

- يتم عبر `user_identifiers` (Upsert) بدل تعديل أعمدة في `users`.
- Use-cases:
  - `app/Services/UserManagement/Identifiers/UpsertUserIdentifier.php`
  - `app/Services/UserManagement/Identifiers/SetPrimaryIdentifier.php`

### 4) تحديث حالة التحقق للبريد/الجوال

- يتم عبر `user_identifiers.verified_at`.
- Use-case:
  - `app/Services/UserManagement/Identifiers/MarkIdentifierVerified.php`

### 5) تغيير كلمة المرور

- يتم عبر `user_credentials`.
- Use-case:
  - `app/Services/UserManagement/Credentials/UpdateUserPassword.php`

> الفهرس الكامل: `docs/services/user-management/tech/service-functions.md`

