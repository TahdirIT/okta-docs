---
title: نموذج البيانات — ExamAttendance
---

## الغرض
Pivot يُسجّل حضور الطالب لجلسة معينة (ExamDayPeriod) داخل فترة محددة. يدعم حالات الحضور والتأخر والغياب ويحتفظ بوقت التسجيل.

## الحقول المقترحة
- id: مفتاح أساسي (أو مفتاح مركب على student_id + exam_day_period_id).
- exam_period_id: مرجع إلى `ExamPeriod` (لتجميع التقارير بحسب الفترة).
- exam_day_period_id: مرجع إلى `ExamDayPeriod`.
- student_id: مرجع إلى الطالب.
- status (enum): PRESENT, ABSENT, LATE_LESS_THAN_15, LATE_MORE_THAN_15.
- attendance_time (datetime): وقت تسجيل الحالة.
- created_at, updated_at.

## العلاقات
- belongsTo ExamPeriod.
- belongsTo ExamDayPeriod.
- belongsTo Student.

## الفهارس والقيود
- فهرس مركّب على (exam_day_period_id, student_id).
- فهرس على (exam_period_id).
- قيد فريد يمنع تسجيل أكثر من سجل نشط لنفس الطالب والجلسة.

## ملاحظات أعمال
- يمكن للمراقب تحديث الحالة أثناء الجلسة.
- يعتمد تحديد "متأخر" على مقارنة وقت التسجيل مع `ExamDayPeriod.start_time`.

