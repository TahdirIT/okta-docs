---
title: نموذج البيانات — ExamSubject
---

## الغرض
تعريف مادة اختبار مجدولة ضمن فترة يوم محددة (ExamDayPeriod)، مع ربطها بالصف ومجموعة من الفصول (الشعب) المستهدفة.

## الجدول: `exam_subjects`

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `id` | bigint PK | مفتاح أساسي |
| `exam_day_period_id` | FK → exam_day_periods | الفترة الزمنية التي تُجرى فيها المادة |
| `tenant_subject_id` | unsignedBigInteger | مرجع cross-DB إلى `tenant_subjects` في قاعدة البيانات الرئيسية |
| `name_ar` | string nullable | تجاوز الاسم العربي (إذا تُرك فارغاً يُستخدم اسم TenantSubject) |
| `name_en` | string nullable | تجاوز الاسم الإنجليزي |
| `end_time` | time nullable | وقت نهاية المادة (موروث، لا يُستخدم في الواجهة الجديدة) |
| `duration_minutes` | smallint unsigned nullable | مدة الاختبار بالدقائق |
| `created_at`, `updated_at` | timestamps | |

## جدول الفصول: `exam_subject_sections`

جدول pivot يربط كل مادة اختبار بالفصول (الشعب) المستهدفة.

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `id` | bigint PK | |
| `exam_subject_id` | FK → exam_subjects (cascade delete) | |
| `tenant_section_id` | unsignedBigInteger | مرجع cross-DB إلى `tenant_sections` |
| `created_at`, `updated_at` | timestamps | |

- قيد unique: `[exam_subject_id, tenant_section_id]`
- فهرس على `tenant_section_id`

## العلاقات
- `belongsTo(ExamDayPeriod)` عبر `exam_day_period_id`
- `hasMany(ExamSubjectSection)` ← الفصول المرتبطة
- `hasMany(ExamAttendance)` ← سجلات الحضور

## قواعد الأعمال

### تعارض الفصول
لا يمكن أن يكون لنفس الفصل (tenant_section_id) أكثر من مادة اختبار واحدة في نفس الفترة اليومية (exam_day_period_id).
يتم التحقق من ذلك في:
- `Services/Subjects/CreateExamSubject.php`
- `Services/Subjects/UpdateExamSubject.php`

عند وجود تعارض يُرمى `ValidationException` برسالة تحتوي اسم الفصل المتعارض.

### المدة vs وقت النهاية
الواجهة تستخدم `duration_minutes` (مدة بالدقائق) بدلاً من `end_time`.
وقت النهاية الفعلي = `start_time` للفترة + `duration_minutes`.

## الخصائص المحسوبة
- `display_name_ar`: يعيد `name_ar` إذا وُجد، وإلا يرجع إلى `TenantSubject.name_ar`
- `display_name_en`: يعيد `name_en` إذا وُجد، وإلا يرجع إلى `TenantSubject.name_en`

## ملاحظات
- `tenant_subject_id` و`tenant_section_id` مراجع cross-DB — لا توجد FK constraint في قاعدة بيانات `exams_management`.
- تُرتّب المواد للطالب بحسب تاريخ اليوم والفترة الزمنية عند العرض.
