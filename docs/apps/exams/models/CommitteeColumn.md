---
title: نموذج البيانات — CommitteeColumn
---

## الغرض
تعريف عمود داخل لجنة (يساعد على تنظيم المقاعد عموديًا، وقد يُستخدم لتجميع حسب مرحلة).

## الحقول المقترحة
- id: مفتاح أساسي.
- committee_id: مرجع إلى `Committee`.
- order (int): ترتيب العمود داخل اللجنة.
- created_at, updated_at, deleted_at.

## العلاقات
- belongsTo Committee.
- hasMany CommitteeTable: مقاعد/طاولات ضمن هذا العمود.
- belongsToMany EducationLevel (اختياري): في حال دعم تمييز الأعمدة بمراحل.

## الفهارس والقيود
- فهرس مركّب على (committee_id, order).
- الحفاظ على تسلسل فريد لـ order داخل اللجنة.

