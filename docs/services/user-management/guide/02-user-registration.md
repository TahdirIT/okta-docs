# تسجيل مستخدم (User Registration)

هذا المستند يصف تدفق **تسجيل مستخدم جديد** ضمن خدمة `user-management`.

## متى نستخدم هذا التدفق؟

- تسجيل مستخدم بشكل مباشر (إن كانت المنصة تتيح ذلك).
- ضمن تدفقات خدمات أخرى مثل `tenant-registration` عند إنشاء المستخدم المالك (Owner).

## المدخلات المطلوبة

الحد الأدنى المقترح:

- `name`
- `password` + `password_confirmation`

والحقول التعريفية (حسب سياسة المنتج/الدولة):

- `username?`
- `national_id?`
- `mobile?`
- `email?`

## قواعد أساسية

- تطبيق سياسة كلمات المرور في المنصة.
- منع التكرار (Uniqueness) للحقول: `national_id/username/mobile/email`:
  - المرجع: `docs/services/user-management/tech/models/user_identifiers.md`
- لا يتم إسناد أدوار هنا (RBAC منفصل):
  - المرجع: `docs/services/role-based-access-control/tech/permissions.md`

## الـ Service (Use-case) المقترح في Laravel

- `app/Services/UserManagement/Users/RegisterUser.php`

> تفاصيل قائمة الوظائف: `docs/services/user-management/tech/service-functions.md`

