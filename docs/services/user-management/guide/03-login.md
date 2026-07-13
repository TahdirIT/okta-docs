# تسجيل الدخول (Login)

هذا المستند يصف تدفق **تسجيل الدخول** ضمن خدمة `user-management`.

## المدخلات

- `identifier`: اسم المستخدم أو البريد أو الجوال (حسب سياسة تسجيل الدخول)
- `password`

## قواعد أساسية

- Rate limiting لمحاولات الدخول الفاشلة.
- رسائل خطأ عامة (بدون كشف إن كان الحساب موجودًا).
- عند نجاح الدخول: إنشاء/تحديث Session (token/cookie) حسب أسلوب المصادقة في المنصة.

## الـ Services (Use-cases) المقترحة في Laravel

- `app/Services/UserManagement/Auth/LoginUser.php`
- `app/Services/UserManagement/Auth/LogoutUser.php`
- `app/Services/UserManagement/Auth/RefreshSession.php` (إن كان أسلوب المصادقة يدعم refresh)

> تفاصيل قائمة الوظائف: `docs/services/user-management/tech/service-functions.md`

