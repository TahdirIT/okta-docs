# مراجع تخصيصات تسجيل الكيان (Entity Registration Customizations)

> **ملاحظة مهمة**: هذا الملف متعلق ببرمجة **entity-registration-customizations** فقط. جميع المراجع والملفات المذكورة هنا خاصة بهذه الميزة فقط.

هذا الملف يحتوي على جميع المراجع والملفات والمجلدات المتعلقة بميزة **تخصيصات تسجيل الكيان** عند كتابة prompt لأي AI agent.

---

## 📚 التوثيق الشامل (Guide)

**الملف:** `docs/services/contries-management/guide/entity-registration-customizations.md`

يحتوي على:
- بيانات التواصل الإلزامية (جوال/بريد)
- أنواع الكيانات وترتيبها
- تفعيل/تعطيل أنواع الكيانات
- الحقول المخصصة الإضافية حسب نوع الكيان
- أنواع الخانات المدعومة
- مواصفات تعريف الحقل (Schema)
- شروط الظهور (Conditional visibility)
- التحقق/القيود (Validations): `validations` + `regex_rules` (مع رسائل ar/en) + limits (min/max)

---

## 🎨 تجربة المستخدم (User Experience)

**الملف:** `docs/services/contries-management/user-experience/entity-registration-customizations-ux.md`

يحتوي على:
- الوصول إلى الصفحة
- الهدف من الصفحة
- إعدادات بيانات التواصل الإلزامية
- أنواع الكيانات (عرض وتفعيل/تعطيل)
- الحقول المخصصة الإضافية (إضافة/تعديل/حذف)
- تعدد اللغات (Labels i18n)
- تحققّات وأخطاء شائعة
- سلوك واجهة إعداد `validations` (حدود + Regex متعددة برسائل ar/en)

---

## 💻 التوثيق التقني (Tech)

### معالجة البيانات (Data Handling)

**الملف:** `docs/services/contries-management/tech/data-handling/entity_registration_customizations.md`

يحتوي على:
- الغرض من الإعدادات
- مكان التخزين (جدول `model_settings`)
- الشكل العام (Schema) الكامل
- تفاصيل `contact_requirements`
- تفاصيل `entity_registration.types_order`
- تفاصيل `entity_registration.types`
- تفاصيل `entity_registration.custom_fields`
- تعريف الحقل (Field definition)
- خصائص إضافية حسب النوع
- شروط الظهور (Conditional visibility)
- القيود والتحقق (Validation / Constraints)
- توثيق `validations` (Regex rules + limits) داخل تعريف الحقل

### الصلاحيات (Permissions)

**الملف:** `docs/services/contries-management/tech/permissions.md`

**القسم:** تخصيصات تسجيل الكيان (Entity Registration Customizations)

#### الصلاحيات المطلوبة:
- `countries_management.entity_registration_customizations.view` - عرض إعدادات تخصيصات تسجيل الكيان
- `countries_management.entity_registration_customizations.update` - تعديل إعدادات تخصيصات تسجيل الكيان
- `countries_management.entity_registration_customizations.entity_types.update` - تفعيل/تعطيل أنواع الكيانات أثناء التسجيل
- `countries_management.entity_registration_customizations.custom_fields.view` - عرض الحقول المخصصة حسب نوع الكيان
- `countries_management.entity_registration_customizations.custom_fields.create` - إضافة حقل مخصص
- `countries_management.entity_registration_customizations.custom_fields.update` - تعديل حقل مخصص
- `countries_management.entity_registration_customizations.custom_fields.delete` - حذف حقل مخصص

#### القواعد:
- **بيانات التواصل الإلزامية**: يمكن جعل (الجوال/البريد) كلاهما إلزامي أو أحدهما أو كلاهما اختياري حسب سياسة الدولة
- **تفعيل/تعطيل أنواع الكيانات**: تعطيل نوع كيان يعني عدم ظهوره للمستخدم النهائي في خطوة اختيار نوع الكيان أثناء التسجيل، تعطيل نوع كيان لا يحذف الحقول المخصصة التابعة له، ترتيب أنواع الكيانات يبقى ثابتًا كما هو في `docs/roles-and-entities/tenants.md`
- **الحقول المخصصة**: يجب التحقق من صحة `key` (يسمح فقط بـ `a-z`, `0-9`, `_`)، يجب منع تكرار `key` ضمن نفس (الدولة + نوع الكيان)، الحقول الاختيارية لا يمكن حفظها بدون `options`، عند تفعيل شرط الظهور يجب أن يكون الشرط مكتملًا، يجب منع حلقات الاعتمادية

### النماذج (Models)

**الملف:** `docs/services/contries-management/tech/models/model_settings.md`

يحتوي على:
- هيكل جدول `model_settings`
- الأعمدة (id, ulid, model_type, model_id, settings, timestamps)
- الفهارس والقيود
- استخدامه لتخزين إعدادات الدولة

---

## 📋 المعايير التقنية العامة

**المجلد:** `docs/tech-standards/`

### الملفات:

#### `README.md`
- الفهرس الرئيسي للمعايير التقنية

#### `design-standards.md`
- معايير الهوية والتصميم
- الألوان (اللون الرئيسي: #80519f)
- الخطوط (IBM Plex Sans Arabic)
- اللغة والاتجاه (ar RTL, en LTR)
- نقاط توقف الواجهات (Breakpoints)
- المكونات (Components)

#### `permissions-naming.md`
- معيار تسمية أكواد الصلاحيات
- الصيغة: `<feature>.<resource>.<action>`
- استخدام `snake_case` للأسماء المركبة
- أمثلة صحيحة/خاطئة

#### `laravel-feature-services-structure.md`
- معيار تنظيم Feature Services في Laravel
- الهيكلة: `app/Services/<Feature>/<Resource>/<Function>.php`
- قواعد التسمية (PascalCase)
- شكل الملف (Template)
- كيفية الاستخدام

---

## 📄 الملفات العامة في المشروع

**المجلد:** `docs/`

### `README.md`
- نظرة عامة على المشروع
- الخدمات الأساسية
- الفئات المستهدفة
- ميزات الاشتراك
- الستايل والهوية البصرية

### `roles-and-entities/tenants.md`
- **مهم جداً**: يحتوي على أنواع الكيانات التعليمية وترتيبها:
  1. معلم فردي
  2. مدرسة إدارية
  3. مجمع
  4. كلية
  5. جامعة
  6. معهد
  7. أكاديمية
  8. شركة تعليمية
- يجب أن تطابق العناوين المعروضة للمستخدم العناوين في هذا الملف

### `roles-and-entities/end-users.md`
- مجموعات End-User (مسؤول الحساب، إداري، معلم، طالب، ولي أمر، مفوض ولي أمر)

### `packages.md`
- الحزم المستخدمة في المشروع

### `partner-platform.md`
- معلومات عن منصة الشراكات

---

## 🎯 كيفية استخدام هذه المراجع

عند كتابة prompt لأي AI agent للعمل على ميزة تخصيصات تسجيل الكيان:

1. **للتوثيق الشامل:** ابدأ بقراءة `guide/entity-registration-customizations.md`
2. **لتجربة المستخدم:** راجع `user-experience/entity-registration-customizations-ux.md`
3. **للنماذج والبنية التقنية:** راجع `tech/data-handling/entity_registration_customizations.md` و `tech/models/model_settings.md`
4. **للصلاحيات:** راجع `tech/permissions.md` (القسم الخاص بتخصيصات تسجيل الكيان)
5. **للمعايير:** راجع `tech-standards/` للتأكد من اتباع المعايير
6. **لأنواع الكيانات:** راجع `roles-and-entities/tenants.md` لفهم أنواع الكيانات وترتيبها

---

## 📝 ملاحظات مهمة

- جميع الملفات مكتوبة باللغة العربية
- المشروع يستخدم Laravel مع Livewire (بدون API)
- يتم استخدام ULID للجداول التي تحتاجها
- يتم استخدام Soft Deletes حيث مطلوب
- يجب اتباع معايير التسمية في `tech-standards/permissions-naming.md`
- **التخزين**: يتم تخزين إعدادات تخصيصات تسجيل الكيان في جدول `model_settings` تحت مفتاح `settings.entity_registration_customizations`
- **أنواع الكيانات**: يجب أن تطابق العناوين المعروضة للمستخدم العناوين في `docs/roles-and-entities/tenants.md`
- **تعدد اللغات**: يجب دعم تعدد اللغات في `label` و `options[].label` (ar, en)
- **شروط الظهور**: يجب التحقق من اكتمال الشرط ومنع حلقات الاعتمادية
- **validations**:
  - يمكن لكل حقل تعريف `validations` (اختياري) للتحقق/القيود.
  - **Multiple Regex verifications**: استخدم `validations.regex_rules[]`، ولكل قاعدة:
    - `regex`
    - `message.ar` و `message.en` لرسائل الخطأ
  - **Limits validations** (حد أدنى/أعلى) حسب النوع داخل `validations`:
    - نص: `min_length`, `max_length`
    - رقم: `min`, `max`
    - تاريخ: `min_date`, `max_date` (ISO `YYYY-MM-DD`)
    - ملف: `min_size_mb` (اختياري), `max_size_mb` (+ `accept` يبقى على مستوى الحقل)
    - قوائم متعددة (اختياري حسب المنتج): `min_selected`, `max_selected`
  - عند وجود `regex_rules` تُطبّق بالترتيب؛ **أول قاعدة تفشل تُرجع رسالتها** (حسب لغة الواجهة).
  - يجب منع إدخال حدود غير منطقية (مثال: `min > max`، `min_length > max_length`، `min_date > max_date`).

---

## 🔗 الترتيب المقترح للقراءة

1. `guide/entity-registration-customizations.md` - لفهم الميزة بشكل عام
2. `user-experience/entity-registration-customizations-ux.md` - لفهم تجربة المستخدم
3. `tech/data-handling/entity_registration_customizations.md` - لفهم البنية التقنية والـ Schema
4. `tech/models/model_settings.md` - لفهم مكان التخزين
5. `tech/permissions.md` - لفهم الصلاحيات المطلوبة
6. `roles-and-entities/tenants.md` - لفهم أنواع الكيانات
7. `tech-standards/` - لفهم المعايير التقنية

---

## 🤖 إضافة جاهزة للاستخدام في Claude (Prompt snippet)

تم نقل الإضافة إلى ملف مستقل داخل `docs/prompts/`:

- `docs/prompts/add-entity-registration-customizations-validations.md`
