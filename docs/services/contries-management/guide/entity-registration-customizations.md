# تخصيصات تسجيل الكيان (ضمن مزايا إعدادات الدولة)

هذا القسم يوضح إعدادات “تخصيصات تسجيل الكيان” التي تُدار **على مستوى الدولة** وتؤثر على تجربة إنشاء/تسجيل كيان جديد (Tenant) في المنصة.

## 1) بيانات التواصل/الهوية الإلزامية (لكل الكيانات)

تحدد الدولة متطلبات البيانات الأساسية أثناء تسجيل أي كيان:

- **إلزام اسم المستخدم (Username)**: تحديد ما إذا كان إدخال اسم المستخدم إجباريًا عند **إنشاء حساب مالك جديد** ضمن تدفق التسجيل.
- **إلزام رقم الجوال**: تحديد ما إذا كان إدخال رقم الجوال إجباريًا.
- **إلزام البريد الإلكتروني**: تحديد ما إذا كان إدخال البريد الإلكتروني إجباريًا.
- **إلزام رقم الهوية الوطنية**: تحديد ما إذا كان إدخال رقم الهوية الوطنية إجباريًا.

> ملاحظة:
> - يمكن أن تكون أي تركيبة مطلوبة حسب سياسة الدولة (اسم المستخدم/الجوال/البريد/الهوية)، بما في ذلك إلزام حقل واحد أو أكثر أو جعلها اختيارية.
> - في مسار “لدي حساب مسبق” لا يتم “إنشاء” اسم مستخدم جديد؛ بدلاً من ذلك يُستخدم مُعرّف الدخول (قد يكون اسم المستخدم/الجوال/البريد حسب ما تدعمه المصادقة).

### 1.1) تحقق Regex للجوال ورقم الهوية حسب الدولة (Country-specific validation)

بالإضافة إلى “الإلزامية”، قد تحتاج الدولة إلى فرض **قواعد تحقق مختلفة** على:

- **رقم الجوال** (Mobile)
- **رقم الهوية الوطنية** (National ID)

لذلك يمكن تعريف إعدادات تحقق على مستوى الدولة (مقترح) تحت كائن مثل:

- `contact_validations.mobile`
- `contact_validations.national_id`

وبنفس أسلوب `regex_rules` المستخدم في الحقول المخصصة:

- تطبيق القواعد بالترتيب.
- أول Regex يفشل يُرجع رسالة الخطأ (`ar/en`) حسب لغة الواجهة.

مثال تمثيلي (سعودي) — *القيم Regex مجرد أمثلة قابلة للتغيير حسب سياسة كل دولة*:

```json
{
  "contact_requirements": {
    "username_required": true,
    "national_id_required": true,
    "mobile_required": true,
    "email_required": false
  },
  "contact_validations": {
    "mobile": {
      "regex_rules": [
        {
          "regex": "^05[0-9]{8}$",
          "message": { "ar": "رقم الجوال يجب أن يبدأ بـ 05 ويتكون من 10 أرقام", "en": "Mobile must start with 05 and be 10 digits" }
        }
      ]
    },
    "national_id": {
      "regex_rules": [
        {
          "regex": "^[12][0-9]{9}$",
          "message": { "ar": "رقم الهوية يجب أن يكون 10 أرقام ويبدأ بـ 1 أو 2", "en": "National ID must be 10 digits and start with 1 or 2" }
        }
      ]
    }
  }
}
```

## 2) أنواع الكيانات (مرتبة حسب `docs/groups/tenants.md`)

يتم عرض أنواع الكيانات للمستخدم بالترتيب التالي:

1. **معلم فردي**
2. **مدرسة إدارية**
3. **مجمع**
4. **كلية**
5. **جامعة**
6. **معهد**
7. **أكاديمية**
8. **شركة تعليمية**

### 2.1) تفعيل/تعطيل نوع كيان

بالإضافة إلى الترتيب الثابت، يمكن للدولة **تفعيل أو تعطيل** ظهور نوع كيان للمستخدم أثناء التسجيل:

- النوع **المعطّل** لا يظهر للمستخدم النهائي في خطوة اختيار نوع الكيان.
- تعطيل النوع **لا يحذف** إعدادات الحقول المخصصة التابعة له؛ تبقى محفوظة ويمكن إعادة تفعيله لاحقًا.

## 3) الحقول المخصصة الإضافية حسب نوع الكيان

بعد اختيار نوع الكيان، يمكن للدولة تعريف **حقول إضافية** تظهر في نموذج التسجيل لهذا النوع (أو لجميع الأنواع).

### ما الذي نُقصده بـ “حقل مخصص”؟

هو خانة إضافية داخل نموذج تسجيل الكيان، تُعرّفها الدولة لتجميع بيانات مطلوبة أو اختيارية، مع دعم:

- نوع الخانة
- مفتاح (Key) ثابت يُستخدم في التخزين/الواجهة البرمجية
- عنوان الخانة (Label) الظاهر للمستخدم
- خيارات الخانة (إن كانت من نوع اختيارات)
- شروط الظهور بناءً على اختيار حقول سابقة
- إلزامية الإدخال عند الظهور

### 3.1) أنواع الخانات المقترحة

يمكن دعم الأنواع التالية (مع قابلية التوسع لاحقًا):

- **text**: نص قصير
- **textarea**: نص طويل
- **number**: رقم
- **date**: تاريخ
- **email**: بريد إلكتروني
- **phone**: رقم جوال/هاتف
- **url**: رابط
- **select**: قائمة اختيار واحدة (Dropdown)
- **multi_select**: قائمة متعددة الاختيارات
- **radio**: اختيار واحد (Radio)
- **checkbox**: اختيار/تفعيل (Boolean)
- **checkbox_group**: مجموعة Checkboxes (متعددة)
- **file**: رفع ملف (مع قيود نوع/حجم)

> اقتراح اختياري حسب احتياج الدولة: **commercial_registration** كأنواع متخصصة مع قواعد تحقق إضافية.

### 3.2) مواصفات تعريف الحقل (Schema)

الحقول المخصصة يمكن تمثيلها بهذه الخصائص:

1. **نوع الخانة** (`type`)
2. **key** (`key`)
3. **عنوان الخانة متعدد اللغات** (`label`)
4. **خيارات الخانة** (`options`) بحسب النوع
5. **شرط الظهور** (`visibility`) هل هو مشروط أم مفتوح
6. **إجباري عند الظهور** (`required_when_visible`)
7. **التحقق/القيود** (`validations`) (اختياري)

#### تعدد اللغات في `label`

- كل `label` يجب أن يدعم عدة لغات (مثال: العربية/الإنجليزية)، لذلك يُمثّل ككائن (Object) مثل:
  - `{ "ar": "…", "en": "…" }`
- نفس الفكرة تنطبق على عناوين الخيارات داخل `options[].label`.

#### توصيات على `key`

- يكون **فريدًا** ضمن سياق الدولة + نوع الكيان.
- بصيغة `snake_case` مثل: `license_number`, `school_gender`, `has_boarding`.
- يُفضّل عدم تغييره بعد الاستخدام لتجنب كسر البيانات التاريخية.

#### 3.2.1) التحقق (Validations) — دعم عدة Regex

يمكن إضافة كائن `validations` لتعريف قيود التحقق على قيمة الحقل. ضمنه يمكن دعم:

- تحقق Regex واحد بسيط عبر `pattern` (للاحتياج البسيط/التوافقية).
- **تحقق Regex متعدد** عبر `regex_rules`، بحيث يمكن تعريف أكثر من Regex، ولكل واحد **رسالة خطأ عربية وإنجليزية**.
- **قيود حدود (Limits)** مثل حد أدنى/أعلى حسب نوع الحقل (نص/رقم/تاريخ/ملف...).

صيغة مقترحة:

```json
{
  "validations": {
    "pattern": "^[0-9]{10}$",
    "regex_rules": [
      {
        "regex": "^[0-9]{10}$",
        "message": { "ar": "رقم الترخيص يجب أن يكون 10 أرقام", "en": "License number must be 10 digits" }
      },
      {
        "regex": "^(?!0000000000).*$",
        "message": { "ar": "رقم الترخيص غير صالح", "en": "Invalid license number" }
      }
    ]
  }
}
```

ملاحظات:

- `regex_rules` تُطبق بالترتيب؛ أول قاعدة تفشل تُرجع رسالتها.
- `pattern` و `regex_rules` اختياريان؛ ويمكن دعم أحدهما أو كليهما حسب الاحتياج.
- عند وجود `regex_rules` يُفضّل إهمال/تجاهل `pattern` في التنفيذ الفعلي لتجنب ازدواجية التحقق (قرار تنفيذي).

#### 3.2.2) Limits validations (حد أدنى/أعلى) حسب النوع

داخل `validations` يمكن تعريف حدود دنيا/عليا بطريقة مناسبة لنوع الحقل:

- **text / textarea**:
  - `min_length`, `max_length`
- **number**:
  - `min`, `max`
- **date**:
  - `min_date`, `max_date` (صيغة ISO مثل `YYYY-MM-DD`)
- **file**:
  - `max_size_mb`
  - (اختياري) `min_size_mb`
- **select / multi_select / checkbox_group** (اختياري حسب المنتج):
  - `min_selected`, `max_selected`

مثال تمثيلي (جزء من حقل):

```json
{
  "validations": {
    "min_length": 3,
    "max_length": 50,
    "min": 1,
    "max": 9999,
    "min_date": "2020-01-01",
    "max_date": "2030-12-31",
    "min_selected": 1,
    "max_selected": 3,
    "min_size_mb": 0.1,
    "max_size_mb": 10
  }
}
```

### 3.3) خيارات الخانة (حسب النوع)

- **select / multi_select / radio / checkbox_group**:
  - `options`: قائمة قيم مثل:
    - `value`: قيمة تُخزن
    - `label`: عنوان الخيار متعدد اللغات يظهر للمستخدم
- **file**:
  - `accept`: الامتدادات المسموحة (مثل: `pdf`, `png`, `jpg`)
  - (القيود مثل `max_size_mb` تكون داخل `validations`)
- **ملاحظة**:
  - القيود/التحقق مثل `min_length`, `max_length`, `min`, `max`, `pattern`, `regex_rules` تكون داخل `validations` (إن لزم).

## 4) شروط الظهور (Conditional visibility)

يمكن للحقل أن يكون:

- **مفتوح بشكل عام**: يظهر دائمًا بعد اختيار نوع الكيان.
- **مشروط**: يظهر فقط إذا تحقق شرط مبني على قيم حقول سابقة.

### 4.1) أمثلة لشروط منطقية مقترحة

- **إذا تم تفعيل checkbox**:
  - مثال: إظهار حقل `boarding_capacity` فقط إذا كان `has_boarding = true`.
- **إذا تم اختيار قيمة من radio/select**:
  - مثال: إظهار `girls_section_details` إذا كان `school_gender = girls`.
- **إذا كانت القيمة ضمن مجموعة**:
  - مثال: إظهار `license_file` إذا كانت `entity_type` ضمن (جامعة/كلية).

### 4.2) صيغة شرط مقترحة (مثال تمثيلي)

```json
{
  "field": "school_gender",
  "operator": "equals",
  "value": "girls"
}
```

اقتراحات للـ `operator`:

- `equals`, `not_equals`
- `in`, `not_in`
- `is_true`, `is_false`
- `exists` (عند وجود قيمة غير فارغة)

## 5) مثال توضيحي كامل (تمثيل مقترح)

المثال التالي يوضح تعريف حقول مخصصة لنوع “مدرسة إدارية”:

```json
{
  "contact_requirements": {
    "username_required": true,
    "national_id_required": true,
    "mobile_required": true,
    "email_required": false
  },
  "contact_validations": {
    "mobile": {
      "regex_rules": [
        {
          "regex": "^05[0-9]{8}$",
          "message": { "ar": "رقم الجوال يجب أن يبدأ بـ 05 ويتكون من 10 أرقام", "en": "Mobile must start with 05 and be 10 digits" }
        }
      ]
    },
    "national_id": {
      "regex_rules": [
        {
          "regex": "^[12][0-9]{9}$",
          "message": { "ar": "رقم الهوية يجب أن يكون 10 أرقام ويبدأ بـ 1 أو 2", "en": "National ID must be 10 digits and start with 1 or 2" }
        }
      ]
    }
  },
  "entity_registration": {
    "types_order": [
      "individual_teacher",
      "administrative_school",
      "complex",
      "college",
      "university",
      "institute",
      "academy",
      "education_company"
    ],
    "custom_fields": {
      "administrative_school": [
        {
          "type": "text",
          "key": "license_number",
          "label": { "ar": "رقم الترخيص", "en": "License number" },
          "visibility": { "mode": "always" },
          "required_when_visible": true,
          "validations": {
            "regex_rules": [
              {
                "regex": "^[0-9]{10}$",
                "message": { "ar": "رقم الترخيص يجب أن يكون 10 أرقام", "en": "License number must be 10 digits" }
              },
              {
                "regex": "^(?!0000000000).*$",
                "message": { "ar": "رقم الترخيص غير صالح", "en": "Invalid license number" }
              }
            ]
          }
        },
        {
          "type": "radio",
          "key": "school_gender",
          "label": { "ar": "نوع المدرسة", "en": "School type" },
          "options": [
            { "value": "boys", "label": { "ar": "بنين", "en": "Boys" } },
            { "value": "girls", "label": { "ar": "بنات", "en": "Girls" } },
            { "value": "mixed", "label": { "ar": "مختلطة", "en": "Mixed" } }
          ],
          "visibility": { "mode": "always" },
          "required_when_visible": true
        },
        {
          "type": "checkbox",
          "key": "has_boarding",
          "label": { "ar": "يوجد سكن داخلي", "en": "Has boarding" },
          "visibility": { "mode": "always" },
          "required_when_visible": false
        },
        {
          "type": "number",
          "key": "boarding_capacity",
          "label": { "ar": "سعة السكن الداخلي", "en": "Boarding capacity" },
          "visibility": {
            "mode": "conditional",
            "when": { "field": "has_boarding", "operator": "is_true" }
          },
          "required_when_visible": true,
          "validations": { "min": 1, "max": 5000 }
        }
      ]
    }
  }
}
```

> ملاحظة: أسماء القيم داخل `types_order` أعلاه تمثل **مفاتيح داخلية مقترحة** (ليس شرطًا أن تكون نهائية)، بينما “عناوين الأنواع” المعروضة للمستخدم يجب أن تطابق العناوين العربية في `docs/groups/tenants.md`.

