---
title: نموذج البيانات — ExamObserver
---

## الغرض
تمثيل تكليف مراقب (معلم) بجلسة محددة: يوم + فترة يوم + لجنة. يتيح له رصد الحضور وإدارة الجلسة.

## الحقول المقترحة
- id: مفتاح أساسي.
- employee_id / teacher_id: مرجع إلى المعلم (كيان الموظف في المنصة).
- exam_day_period_id: مرجع إلى `ExamDayPeriod`.
- committee_id: مرجع إلى `Committee`.
- created_at, updated_at, deleted_at.

## العلاقات
- belongsTo Employee/Teacher.
- belongsTo ExamDayPeriod.
- belongsTo Committee.

## الفهارس والقيود
- فهرس مركّب على (exam_day_period_id, committee_id, employee_id).
- قيد منطقي يمنع إسناد نفس المراقب لأكثر من لجنة في نفس الفترة الزمنية.

