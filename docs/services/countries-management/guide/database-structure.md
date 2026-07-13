# Database structure (Laravel 12 + PostgreSQL)

هذا القسم يوضح **هيكل قاعدة البيانات المقترح** لخدمة **إدارة الدول** مع افتراض:

- **Laravel 12**
- **PostgreSQL** (مع استخدام `jsonb` للترجمات/الإعدادات)
- **PostGIS** اختيارياً لدعم حدود المناطق/الإحداثيات (إن احتجنا `geometry`)

> الهدف: فصل بيانات الدولة المرجعية (Country-level master data) عن بيانات المدرسة، مع توفير إمكانية **override** على مستوى المدرسة عند الحاجة بدون تكرار.

> **ملاحظة حول حزمة `nnjeim/world`:** المشروع يستخدم حزمة `nnjeim/world` لتوفير البيانات الجغرافية الأساسية (الدول، الولايات/المناطق، المدن). يتم استيراد البيانات من البكج ثم إضافة الحقول الإضافية المطلوبة للمشروع. راجع ملفات النماذج في `tech/models/` للتفاصيل الكاملة.

## الجداول الأساسية (Country-level)

### 1) `countries`

مرجع الدولة وإعداداتها العامة.

> **يعتمد على:** حزمة `nnjeim/world` - جدول `countries`. يتم استيراد البيانات الأساسية من البكج (مثل: `name`, `iso2`, `iso3`, `phone_code`, `native`, `region`, `subregion`, `latitude`, `longitude`, `emoji`) ثم إضافة الحقول الإضافية (مثل: `ulid`, `name_ar`, `name_en`, `name_json`, `active`, `deleted_at`).

### 2) `regions`, `cities`, `districts`

الهيكل الجغرافي للدولة (مناطق ← مدن ← أحياء).

> **يعتمد على:** 
> - **`regions`**: حزمة `nnjeim/world` - جدول `states` (الولايات/المحافظات في البكج = المناطق في المشروع)
> - **`cities`**: حزمة `nnjeim/world` - جدول `cities`
> - **`districts`**: خاص بالمشروع - البكج لا يوفر بيانات الأحياء

### 3) `model_settings` (أو بديل صريح)

تخزين إعدادات مرتبطة بالنماذج، أهمها في سياق إدارة الدول:

- `weekdays`
- `active_term_id`

> في مشروع `v5website` هذه الإعدادات محفوظة داخل `model_settings.settings` كـ JSON. في Postgres يُفضّل أن تكون `jsonb` مع فهارس مناسبة.

## التقويم (Calendar)

### 4) `academic_years`

إطار زمني على مستوى الدولة يحتوي على فصول (Terms).

### 5) `terms`

فصول مرتبطة بعام دراسي (`academic_year_id`) مع نطاق زمني.

### 6) `holiday_schedules`

جدول الإجازات/الأنشطة، بنمط **polymorphic scope** على مستوى الدولة أو المدرسة:

- `related_type`, `related_id` (مرجع: Country أو School)

> في PostgreSQL لا يمكن فرض FK مباشرةً على polymorphic؛ نعوض ذلك بـ indexes وقواعد تطبيق (Application-level constraints).

## الهيكل الأكاديمي (Stages / Levels / Subjects)

### 7) `education_level_groups`

مجموعات المراحل (مثل: ابتدائي/متوسط/ثانوي) على مستوى الدولة.

### 8) `education_levels` (مقترح)

مستويات/صفوف الدولة (KG-1, Grade 1, …) مع ترتيب وخصائص (مثل العمر المتوقع).

### 9) `education_level_group_levels` (مقترح Pivot)

ربط المستويات بمجموعاتها.

### 10) `subjects` (مُعاد تصميمه لخدمة إدارة الدول)

تعريف مواد على مستوى الدولة، مع إمكانية ربطها بـ:

- مستوى/عدة مستويات (`education_levels`) أو مجموعة مستويات (`education_level_groups`)
- فصل (Term) إن كانت المواد “تتغير” حسب العام/الفصل

> ملاحظة: في `v5website` جدول `subjects` مرتبط بـ `term_id` و `group_level_id` ويحتوي أيضًا `school_id` كـ override. في خدمة إدارة الدول نقترح الفصل الواضح بين تعريف المادة المرجعي على مستوى الدولة وبين تخصيصات المدرسة.

## مقارنة سريعة مع `v5website` (ما الذي سنحافظ عليه؟ وما الذي سنحسّنه؟)

### ما هو مطابق/قريب

- `countries`, `regions`, `cities`, `districts`
- `academic_years`, `terms`
- `holiday_schedules` (polymorphic على Country/School)
- تخزين إعدادات الدولة مثل `weekdays` و `active_term_id` كـ JSON داخل `model_settings`

### التحسينات المقترحة (Laravel 12 + Postgres)

- استخدام **`jsonb`** بدلاً من `json` للترجمات والإعدادات.
- إضافة **قيود** (CHECK) و **فهارس** (Indexes) واضحة:
  - فهارس على `country_id` في كل الجداول التابعة
  - منع التداخل في الإجازات ضمن نفس النطاق (عبر منطق التطبيق +/أو قيود Exclusion في Postgres عند الحاجة)
- اعتماد **PostGIS** إن تم استخدام حدود/مضلعات المناطق (`regions.boundaries`).
- إعادة تصميم stages/levels ليكون على مستوى الدولة:
  - `education_levels` + Pivot بدل ربط `stages` بالمدرسة مباشرةً (كما في `v5website`).

