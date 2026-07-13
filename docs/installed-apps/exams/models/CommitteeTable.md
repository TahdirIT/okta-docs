---
title: نموذج البيانات — CommitteeTable
---

## الغرض
تمثيل مقعد/طاولة في اللجنة، وقد يكون محجوزًا لطالب معيّن أو لمرحلة/صف، مع إمكان تعطيله.

## الحقول المقترحة
- id: مفتاح أساسي.
- committee_column_id: مرجع إلى `CommitteeColumn`.
- order (int): ترتيب المقعد داخل العمود.
- student_id (nullable): مرجع إلى طالب معيّن (إن تم التخصيص الفردي).
- section_id / stage_id (nullable): ربط حسب الفصل/المرحلة عند التوزيع الجماعي.
- reservable (boolean): هل المقعد متاح للحجز/الاستخدام؟
- created_at, updated_at, deleted_at.

## العلاقات
- belongsTo CommitteeColumn.
- belongsTo Student (من نظام المدرسة/المنصة).
- belongsTo Section/Stage (بحسب هيكلة المنصة).

## الفهارس والقيود
- فهرس مركّب على (committee_column_id, order).
- فهارس على (student_id), (section_id), (stage_id) لتحسين تقارير التوزيع.
- قيد منطقي يمنع حجز مقعد غير قابل للحجز.

