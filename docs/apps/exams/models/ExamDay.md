---
title: نموذج البيانات — ExamDay
---

## الغرض
تمثيل يوم اختبار واحد ضمن `ExamPeriod`.

## الحقول المقترحة
- id: مفتاح أساسي.
- exam_period_id: مرجع إلى `ExamPeriod`.
- date: تاريخ اليوم.
- created_at, updated_at, deleted_at.

## العلاقات
- belongsTo ExamPeriod.
- hasMany ExamDayPeriod: الفترات الزمنية ضمن هذا اليوم.

## الفهارس والقيود
- فهرس مركّب على (exam_period_id, date) لضمان تميّز اليوم داخل الفترة.

## ملاحظات أعمال
- ترتيب الأيام بحسب التاريخ مهم لعرض الجداول والتقارير.

