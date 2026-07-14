---
title: نموذج البيانات — ExamDayPeriod
---

## الغرض
تمثيل فترة زمنية ضمن يوم اختبار (مثل: الفترة الأولى الساعة 08:00). ترتبط بها مواد جلسة ذلك اليوم وقوائم النداء والحضور.

## الحقول المقترحة
- id: مفتاح أساسي.
- exam_day_id: مرجع إلى `ExamDay`.
- title: اسم الفترة (اختياري).
- start_time: وقت البدء.
- created_at, updated_at, deleted_at.

## العلاقات
- belongsTo ExamDay.
- hasMany ExamSubject: مواد الجلسة المرتبطة بهذه الفترة.
- belongsToMany Student عبر Pivot للحضور (ExamAttendance).
- hasMany ExamObserver: مراقبو الجلسة (عبر ربط باللجنة).

## الفهارس والقيود
- فهرس مركّب على (exam_day_id, start_time).

## ملاحظات أعمال
- يعتمد تحديد التأخر على مقارنة وقت التسجيل مع `start_time`.

