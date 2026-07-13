# نماذج بيانات إدارة الاختبارات (Models)

> **حالة التوثيق:** خدمة الاختبارات تُقدَّم اليوم كتطبيق مثبَّت **`okta-exams`**
> (راجع [`../overview.md`](../overview.md)). ملف
> [`ExamSubject.md`](./ExamSubject.md) يعكس **البنية الفعلية المُنفَّذة**
> (جداول التطبيق الخاصة، مراجع الجهة بمعرّفات ULID مبهمة **بلا FK** لجداول
> المنصة). أمّا بقية الملفات فهي **مسودّات مفاهيمية سابقة** («الحقول المقترحة»)
> كُتبت قبل بناء التطبيق، وبعضها يصف علاقات `belongsTo`/أعمدة `school_id` تخالف
> [عقد التطبيقات المثبَّتة](../../../../claude/installed-apps.md) (التطبيق يملك
> جداوله ويشير للمنصة بمعرّفات مبهمة فقط). عامِلها كمرجع وظيفي لا كبنية نهائية.

## الملفات

| النموذج | الحالة |
|---|---|
| [`ExamSubject.md`](./ExamSubject.md) | كما هو مُنفَّذ (as-built) |
| [`ExamPeriod.md`](./ExamPeriod.md) | مسودة مفاهيمية |
| [`ExamDay.md`](./ExamDay.md) | مسودة مفاهيمية |
| [`ExamDayPeriod.md`](./ExamDayPeriod.md) | مسودة مفاهيمية |
| [`Committee.md`](./Committee.md) | مسودة مفاهيمية |
| [`CommitteeColumn.md`](./CommitteeColumn.md) | مسودة مفاهيمية |
| [`CommitteeTable.md`](./CommitteeTable.md) | مسودة مفاهيمية |
| [`ExamObserver.md`](./ExamObserver.md) | مسودة مفاهيمية |
| [`ExamAttendance.md`](./ExamAttendance.md) | مسودة مفاهيمية |
