# وظائف المستخدم (Use-cases) — User Management

هذا المستند يوثق **Use-cases** لخدمة **إدارة المستخدمين**، بصيغة متوافقة مع معيار تنظيم Feature Services:

- `docs/tech-standards/laravel-feature-services-structure.md`

صيغة المسارات المقترحة:

`app/Services/UserManagement/<Resource>/<Function>.php`

> ملاحظة: العناوين أدناه تصف ما يفعله المستخدم وظيفيًا، و"مكان الملف" هو مسار مقترح لملف الـService (Use-case class) في Laravel.

## الفهرس

- [1) Registration](#1-registration)
- [2) Login / Sessions](#2-login--sessions)
- [3) User Update](#3-user-update)

---

## 1) Registration

| عنوان الفنكشن | مكان الملف |
|---|---|
| تسجيل مستخدم جديد | `app/Services/UserManagement/Users/RegisterUser.php` |

### تسجيل مستخدم جديد — ملاحظات سلوكية

- **المدخلات (Payload) المقترحة**:
  - `username` (قد يكون إلزاميًا حسب سياق المنتج)
  - `name`
  - `national_id?`
  - `mobile?`
  - `email?`
  - `password`
  - `password_confirmation`
- **قواعد أساسية**:
  - تطبيق سياسة كلمات المرور المعتمدة في المنصة.
  - **منع التكرار (Uniqueness)**: عدم السماح بتسجيل مكرر لـ `national_id/username/mobile/email` (انظر: `models/users.md`).
  - لا يتم إنشاء Roles/Permissions هنا (راجع RBAC).
- **المخرجات (Returns) المقترحة**:
  - `user`
  - (اختياري) `session` إذا كان المنتج يدعم Auto sign-in بعد التسجيل.

> تكامل: في سياق `tenant-registration` قد يتم إنشاء المستخدم المالك (Owner) عبر هذا الـuse-case كجزء من `registerTenant(...)`.

---

## 2) Login / Sessions

| عنوان الفنكشن | مكان الملف |
|---|---|
| تسجيل الدخول | `app/Services/UserManagement/Auth/LoginUser.php` |
| تسجيل الخروج | `app/Services/UserManagement/Auth/LogoutUser.php` |
| تحديث/تدوير التوكن (إن وجد) | `app/Services/UserManagement/Auth/RefreshSession.php` |

### تسجيل الدخول — ملاحظات سلوكية

- **المدخلات (Payload) المقترحة**:
  - `identifier`: اسم المستخدم أو البريد أو الجوال (حسب سياسة تسجيل الدخول)
  - `password`
- **قواعد أساسية**:
  - Rate limiting لمحاولات الدخول الفاشلة.
  - رسائل خطأ عامة (بدون كشف إن كان الحساب موجودًا).
  - يمكن دعم بدائل مستقبلية مثل OTP/SSO دون تغيير مبدأ الهيكلة.
- **المخرجات (Returns) المقترحة**:
  - `user`
  - `session` (token/cookie) حسب أسلوب المصادقة في المنصة

> مرجع RBAC (إسناد الأدوار بعد وجود المستخدم): `docs/services/role-based-access-control/tech/permissions.md`.

---

## 3) User Update

> ملاحظة تصميمية (حسب `okta-web`): تحديث بيانات المستخدم يتم غالبًا عبر ثلاث موارد:
> - `users` (بيانات أساسية مثل `name/status/default_country_id`)
> - `user_identifiers` (username/email/phone/national_id + verified_at + primary)
> - `user_credentials` (password_hash)
>
> مراجع النماذج:
> - `docs/services/user-management/tech/models/users.md`
> - `docs/services/user-management/tech/models/user_identifiers.md`
> - `docs/services/user-management/tech/models/user_credentials.md`

| عنوان الفنكشن | مكان الملف |
|---|---|
| تحديث بيانات المستخدم الأساسية | `app/Services/UserManagement/Users/UpdateUserProfile.php` |
| تغيير حالة المستخدم (تعليق/تفعيل) | `app/Services/UserManagement/Users/UpdateUserStatus.php` |
| تغيير الدولة الافتراضية للمستخدم | `app/Services/UserManagement/Users/UpdateUserDefaultCountry.php` |
| إضافة/تحديث Identifier (email/phone/national_id/username/...) | `app/Services/UserManagement/Identifiers/UpsertUserIdentifier.php` |
| تعيين Identifier أساسي | `app/Services/UserManagement/Identifiers/SetPrimaryIdentifier.php` |
| تحديث حالة التحقق للـ Identifier (تعيين verified_at) | `app/Services/UserManagement/Identifiers/MarkIdentifierVerified.php` |
| تغيير كلمة المرور | `app/Services/UserManagement/Credentials/UpdateUserPassword.php` |
