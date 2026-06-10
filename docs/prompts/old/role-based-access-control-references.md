# مراجع خدمة التحكم في الوصول القائم على الأدوار (Role-Based Access Control)

هذا الملف يحتوي على جميع المراجع والملفات والمجلدات المتعلقة بخدمة **التحكم في الوصول القائم على الأدوار (RBAC)** عند كتابة prompt لأي AI agent.

---

## 📁 البنية الأساسية للمشروع

### المجلد الرئيسي
- `docs/services/role-based-access-control/` - المجلد الرئيسي لخدمة التحكم في الوصول القائم على الأدوار

---

## 📚 التوثيق الشامل (Guide)

**المجلد:** `docs/services/role-based-access-control/guide/`

### الملفات الأساسية:
- `README.md` - الفهرس الرئيسي لملفات التوثيق
- `00-database-structure.md` - هيكل قاعدة البيانات لخدمة RBAC
- `01-overview.md` - نظرة عامة على نظام التحكم في الوصول القائم على الأدوار
- `02-feature-goal.md` - الهدف من الميزة
- `03-roles-permissions.md` - تعريف الأدوار والصلاحيات (System/Tenant) وعلاقتها بالمستخدمين
- `04-permissions-management.md` - إدارة الصلاحيات (settings/permissions)
- `05-roles-management.md` - إدارة الأدوار (settings/roles)
- `06-context-role-switching.md` - السياق (Context) والانتقال بين الأدوار والمستأجرين
- `07-middleware.md` - الوسائط (Middleware) الخاصة بالسياق والتحقق من الصلاحيات
- `08-seeders.md` - البذور (Seeders) الخاصة بالأدوار والصلاحيات والأدوار الثابتة

---

## 💻 التوثيق التقني (Tech)

**المجلد:** `docs/services/role-based-access-control/tech/`

### الملفات الرئيسية:
- `permissions.md` - جميع الصلاحيات المطلوبة لميزة RBAC (rbac.*) والقواعد العامة

### النماذج (Models)
**المجلد:** `docs/services/role-based-access-control/tech/models/`

- `01-permissions.md` - نموذج الصلاحيات (permissions) مع الحقول الإضافية (scope, tenant_id, titles)
- `02-roles.md` - نموذج الأدوار (roles) مع الحقول الإضافية (scope, tenant_id, titles) والأدوار الثابتة
- `03-model_has_roles.md` - نموذج الربط بين المستخدمين والأدوار (model_has_roles) مع عمود tenant_id لتمييز الأدوار على مستوى المستأجر

---

## 🎨 تجربة المستخدم (User Experience)

**المجلد:** `docs/services/role-based-access-control/user-experience/`

### الملفات:
- `README.md` - الفهرس الرئيسي لملفات تجربة المستخدم
- `01-permissions-management-ux.md` - تجربة المستخدم لصفحة إدارة الصلاحيات (`/settings/permissions`)
- `02-roles-management-ux.md` - تجربة المستخدم لصفحة إدارة الأدوار (`/settings/roles`)
- `03-context-selection-ux.md` - تجربة المستخدم لصفحة اختيار السياق والأدوار (`/auth/context`)

---

## 📋 المعايير التقنية العامة

**المجلد:** `docs/tech-standards/`

### الملفات:
- `README.md` - الفهرس الرئيسي للمعايير التقنية
- `design-standards.md` - معايير الهوية والتصميم
- `permissions-naming.md` - معيار تسمية أكواد الصلاحيات (يجب اتباعه عند تعريف صلاحيات RBAC مثل `rbac.permissions.view`)

---

## 📄 الملفات العامة في المشروع

**المجلد:** `docs/`

- `README.md` - نظرة عامة على المشروع
- `roles-and-entities/tenants.md` - معلومات عن المستأجرين (Tenants) وطريقة عمل تعدد المستأجرين
- `roles-and-entities/end-users.md` - معلومات عن المستخدمين النهائيين
- `packages.md` - معلومات عن الحزم (خاصة `spatie/laravel-permission` و/أو `spatie/laravel-multitenancy` إن وُجدت)
- `partner-platform.md` - معلومات عن منصة الشركاء
- `services/core-services.md` - الخدمات الأساسية

---

## 🎯 كيفية استخدام هذه المراجع

عند كتابة prompt لأي AI agent للعمل على خدمة التحكم في الوصول القائم على الأدوار:

1. **للتوثيق الشامل:** ابدأ بقراءة `guide/01-overview.md` ثم `guide/02-feature-goal.md` لفهم الهدف والسياق العام، ثم راجع الملفات الأخرى حسب الحاجة.
2. **للنماذج والبنية التقنية:** راجع `guide/00-database-structure.md` ثم `tech/models/` وملف `tech/permissions.md` لفهم الجداول والعلاقات والأعمدة الإضافية (scope, tenant_id, titles).
3. **لتجربة المستخدم:** راجع `user-experience/` لفهم تدفق الشاشات في `/settings/permissions`, `/settings/roles`, `/auth/context`.
4. **للسياق والوسائط:** راجع `guide/06-context-role-switching.md` و `guide/07-middleware.md` لفهم كيفية بناء السياق واستخدام Middleware والتحكم في `team_id` (إن تم استخدام Teams).
5. **للبذور والأدوار الثابتة:** راجع `guide/08-seeders.md` لفهم طريقة إنشاء الأدوار الثابتة (`superadmin`, `admin`, `teacher`, `student`) وكيفية ربطها بالصلاحيات.
6. **للمعايير:** راجع `tech-standards/permissions-naming.md` للتأكد من أن أسماء الصلاحيات الجديدة متوافقة مع المعايير.

---

## 📝 ملاحظات مهمة

- جميع ملفات خدمة RBAC مكتوبة باللغة العربية.
- المشروع يستخدم Laravel + Livewire، مع حزمة `spatie/laravel-permission` كأساس لإدارة الأدوار والصلاحيات.
- النظام يدعم نطاقين للأدوار والصلاحيات: `system` و `tenant`، مع وجود أدوار ثابتة في كلا النطاقين.
- يتم استخدام Soft Deletes للجداول الحساسة مثل `roles`, `permissions`, `model_has_roles` حيث مطلوب.
- يتم استخدام جدول `model_has_roles` من حزمة `spatie/laravel-permission` لجميع الأدوار، مع إضافة عمود `tenant_id` nullable لتمييز الأدوار على مستوى المستأجر (يكون `null` للأدوار على مستوى النظام)، وإضافة عمود `deleted_at` لدعم Soft Deletes.
- يجب دائماً مسح Cache الصلاحيات عند تغيير سياق المستأجر أو تعديل الأدوار/الصلاحيات (راجع `guide/06-context-role-switching.md` و `guide/07-middleware.md`).
- إذا تم تفعيل Teams في `spatie/laravel-permission`، فإن `team_id` يمثل `tenant_id` ويجب ضبطه عبر Middleware.

---

## 🔗 الترتيب المقترح للقراءة

1. `guide/01-overview.md` - لفهم نظام RBAC بشكل عام.
2. `guide/02-feature-goal.md` - لفهم الهدف من الميزة وسيناريوهات الاستخدام.
3. `guide/00-database-structure.md` - لفهم هيكل قاعدة البيانات (permissions, roles, model_has_roles مع tenant_id، إلخ).
4. `tech/models/` - لفهم النماذج بالتفصيل، خاصة `permissions`, `roles`, `model_has_roles`.
5. `guide/03-roles-permissions.md` - لفهم العلاقة بين الأدوار والصلاحيات والمستخدمين وسياقات system/tenant.
6. `user-experience/` - لفهم تجربة المستخدم لصفحات الإعدادات والسياق.
7. `guide/06-context-role-switching.md` و `guide/07-middleware.md` - لفهم كيفية إدارة السياق والصلاحيات أثناء الطلبات.
8. `tech/permissions.md` - لفهم أكواد الصلاحيات المطلوبة لخدمة RBAC وكيفية تنظيمها.

