# مراجع خدمة إدارة الدول (Countries Management)

هذا الملف يحتوي على جميع المراجع والملفات والمجلدات المتعلقة بخدمة إدارة الدول عند كتابة prompt لأي AI agent.

---

## 📁 البنية الأساسية للمشروع

### المجلد الرئيسي
- `docs/services/contries-management/` - المجلد الرئيسي لخدمة إدارة الدول

---

## 📚 التوثيق الشامل (Guide)

**المجلد:** `docs/services/contries-management/guide/`

### الملفات الأساسية:
- `README.md` - الفهرس الرئيسي لملفات التوثيق
- `database-structure.md` - هيكل قاعدة البيانات الكامل
- `overview.md` - نظرة عامة على الميزة
- `feature-goal.md` - الهدف من الميزة
- `country-data.md` - بيانات الدولة
- `stages-management.md` - إدارة المراحل التعليمية
- `calendar.md` - إدارة التقويم
- `subjects-management.md` - إدارة المواد الدراسية
- `academic-years.md` - إدارة الأعوام الدراسية
- `notifications.md` - إدارة تعريفات الإشعارات
- `entity-registration-customizations.md` - تخصيصات تسجيل الكيان
- `weekdays-settings.md` - إعدادات الأسبوع
- `regions-management.md` - إدارة المناطق

---

## 💻 التوثيق التقني (Tech)

**المجلد:** `docs/services/contries-management/tech/`

### الملفات الرئيسية:
- `permissions.md` - جميع الصلاحيات المطلوبة للميزة
- `__missing-files.md` - الملفات المفقودة التي تحتاج إلى إنشاء

### النماذج (Models)
**المجلد:** `docs/services/contries-management/tech/models/`

- `README.md` - فهرس نماذج البيانات
- `countries.md` - نموذج الدول
- `regions.md` - نموذج المناطق
- `cities.md` - نموذج المدن
- `districts.md` - نموذج الأحياء
- `academic_years.md` - نموذج الأعوام الدراسية
- `terms.md` - نموذج الفصول الدراسية
- `education_level_groups.md` - نموذج مجموعات المراحل التعليمية
- `education_levels.md` - نموذج المراحل التعليمية
- `education_level_group_levels.md` - نموذج ربط المراحل بالمجموعات
- `subjects.md` - نموذج المواد الدراسية
- `subject_education_levels.md` - نموذج ربط المواد بالمراحل
- `holiday_schedules.md` - نموذج جداول الإجازات
- `model_settings.md` - نموذج إعدادات النموذج
- `notification_definitions.md` - نموذج تعريفات الإشعارات
- `notifications.md` - نموذج الإشعارات
- `notification_receivers.md` - نموذج مستقبلي الإشعارات

---

## 🎨 تجربة المستخدم (User Experience)

**المجلد:** `docs/services/contries-management/user-experience/`

### الملفات:
- `README.md` - الفهرس الرئيسي لملفات تجربة المستخدم
- `country-data-ux.md` - تجربة المستخدم لبيانات الدولة
- `stages-management-ux.md` - تجربة المستخدم لإدارة المراحل
- `calendar-ux.md` - تجربة المستخدم للتقويم
- `subjects-management-ux.md` - تجربة المستخدم لإدارة المواد
- `academic-years-ux.md` - تجربة المستخدم للأعوام الدراسية
- `notifications-ux.md` - تجربة المستخدم للإشعارات
- `entity-registration-customizations-ux.md` - تجربة المستخدم لتخصيصات تسجيل الكيان
- `weekdays-settings-ux.md` - تجربة المستخدم لإعدادات الأسبوع
- `regions-management-ux.md` - تجربة المستخدم لإدارة المناطق

---

## ✅ ملاحظة مهمة (تحديثات Entity Registration Customizations)

عند العمل على ميزة `entity-registration-customizations` داخل خدمة إدارة الدول، راعِ أن تعريف الحقول المخصصة يدعم:

- **`validations`** داخل تعريف الحقل (اختياري)
- **Multiple Regex verifications** عبر `validations.regex_rules[]` مع رسائل خطأ `message.ar` و`message.en`
- **Limits validations** (حدود أقل/أعلى) حسب النوع داخل `validations` (مثل `min/max`, `min_length/max_length`, `min_date/max_date`, `max_size_mb`, ...)

المراجع الأساسية:

- `docs/services/contries-management/guide/entity-registration-customizations.md`
- `docs/services/contries-management/user-experience/entity-registration-customizations-ux.md`
- `docs/services/contries-management/tech/data-handling/entity_registration_customizations.md`

إضافة جاهزة للنسخ داخل prompt (Claude):

- `docs/prompts/add-entity-registration-customizations-validations.md`

## 📋 المعايير التقنية العامة

**المجلد:** `docs/tech-standards/`

### الملفات:
- `README.md` - الفهرس الرئيسي للمعايير التقنية
- `design-standards.md` - معايير الهوية والتصميم
- `permissions-naming.md` - معيار تسمية أكواد الصلاحيات

---

## 📄 الملفات العامة في المشروع

**المجلد:** `docs/`

- `README.md` - نظرة عامة على المشروع
- `roles-and-entities/tenants.md` - معلومات عن المستأجرين (Tenants)
- `roles-and-entities/end-users.md` - معلومات عن المستخدمين النهائيين
- `packages.md` - معلومات عن الحزم
- `partner-platform.md` - معلومات عن منصة الشركاء
- `services/core-services.md` - الخدمات الأساسية

---

## 🎯 كيفية استخدام هذه المراجع

عند كتابة prompt لأي AI agent للعمل على خدمة إدارة الدول:

1. **للتوثيق الشامل:** ابدأ بقراءة `guide/overview.md` ثم الملفات الأخرى حسب الحاجة
2. **للنماذج والبنية التقنية:** راجع `tech/models/` و `tech/permissions.md`
3. **لتجربة المستخدم:** راجع `user-experience/` لفهم التدفقات والواجهات
4. **للمعايير:** راجع `tech-standards/` للتأكد من اتباع المعايير

---

## 📝 ملاحظات مهمة

- جميع الملفات مكتوبة باللغة العربية
- المشروع يستخدم Laravel مع Livewire (بدون API)
- يتم استخدام ULID للجداول التي تحتاجها
- يتم استخدام Soft Deletes حيث مطلوب
- يجب اتباع معايير التسمية في `tech-standards/permissions-naming.md`

---

## 🔗 الترتيب المقترح للقراءة

1. `guide/overview.md` - لفهم الميزة بشكل عام
2. `guide/feature-goal.md` - لفهم الهدف من الميزة
3. `guide/database-structure.md` - لفهم هيكل قاعدة البيانات
4. `tech/models/` - لفهم النماذج بالتفصيل
5. `user-experience/` - لفهم تجربة المستخدم
6. `tech/permissions.md` - لفهم الصلاحيات المطلوبة
