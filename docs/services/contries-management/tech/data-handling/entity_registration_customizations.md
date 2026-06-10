# `entity_registration_customizations` (Country-level setting)

## الغرض

تخزين إعدادات **تخصيصات تسجيل الكيان** على مستوى الدولة، والتي تتحكم في:

- متطلبات البيانات الإلزامية أثناء التسجيل (اسم المستخدم/هوية وطنية/جوال/بريد)
- ترتيب عرض **أنواع الكيانات** أثناء التسجيل
- تفعيل/تعطيل ظهور نوع كيان للمستخدم أثناء التسجيل
- تعريف **حقول مخصصة إضافية** تظهر حسب نوع الكيان، مع دعم:
  - عناوين متعددة اللغات (`label`)
  - خيارات متعددة اللغات للحقول الاختيارية (`options[].label`)
  - شروط الظهور (Conditional visibility)
  - إلزامية الإدخال عند الظهور
  - التحقق/القيود (Validations) مثل: عدة Regex مع رسائل خطأ متعددة اللغات، وحدود min/max حسب النوع

> المرجع الوظيفي/المنتجي: `docs/services/contries-management/guide/entity-registration-customizations.md`  
> المرجع UX: `docs/services/contries-management/user-experience/entity-registration-customizations-ux.md`

## أين تُخزّن؟

مثل بقية إعدادات الدولة، يُفضّل تخزينها داخل:

- جدول: `model_settings`
- العمود: `settings` (`jsonb`)
- على موديل الدولة: `model_type = Country`, `model_id = countries.id`

ككائن (Object) تحت مفتاح مقترح:

- `settings.entity_registration_customizations`

## الشكل العام (Schema)

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
    "types": {
      "individual_teacher": { "enabled": true },
      "administrative_school": { "enabled": true },
      "complex": { "enabled": true },
      "college": { "enabled": true },
      "university": { "enabled": true },
      "institute": { "enabled": true },
      "academy": { "enabled": true },
      "education_company": { "enabled": true }
    },
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
        }
      ]
    }
  }
}
```

## التفاصيل

### 1) `contact_requirements`

- **`username_required`**: `boolean` — هل اسم المستخدم (Username) إلزامي عند إنشاء حساب مالك جديد؟
- **`national_id_required`**: `boolean` — هل رقم الهوية الوطنية إلزامي؟
- **`mobile_required`**: `boolean` — هل رقم الجوال إلزامي؟
- **`email_required`**: `boolean` — هل البريد الإلكتروني إلزامي؟

> يمكن أن تكون أي تركيبة إلزامية/اختيارية بين (اسم المستخدم/الهوية الوطنية/الجوال/البريد) حسب سياسة الدولة.

### 1.1) `contact_validations` (اختياري)

دعم تحقق (Validation) على مستوى الدولة للحقول الأساسية:

- `contact_validations.mobile`
- `contact_validations.national_id`

الصيغة المقترحة لكل حقل:

- `regex_rules`: `array`
  - `regex_rules[].regex`: `string`
  - `regex_rules[].message`: `object` (i18n) مثل `{ "ar": "...", "en": "..." }`

ملاحظات:

- تُطبّق القواعد بالترتيب؛ أول قاعدة تفشل تُرجع رسالتها.
- يفضّل أن تكون هذه التحققّات **مكمّلة** لتحقق الصيغة العامة (مثل trimming/normalization) حسب قرار التنفيذ.

### 2) `entity_registration.types_order`

قائمة Strings تمثل ترتيب عرض أنواع الكيانات أثناء التسجيل.

- **ملاحظة**: القيم هنا تمثل **مفاتيح داخلية مقترحة** (slugs)، بينما العناوين المعروضة للمستخدم يجب أن تطابق العناوين في `docs/roles-and-entities/tenants.md`.

### 2.1) `entity_registration.types`

إعدادات لكل نوع كيان (اختياري)، أهمها:

- **`types.{entityType}.enabled`**: `boolean` — هل يظهر هذا النوع للمستخدم النهائي أثناء التسجيل؟

> في حال عدم وجود `types` أو عدم وجود كيان بعينه داخلها، يُفضّل اعتباره **مفعّلًا افتراضيًا**.

### 3) `entity_registration.custom_fields`

كائن مفاتيحه هي **نوع الكيان** (مثل `administrative_school`) وقيمه مصفوفة من تعريفات الحقول.

> (اختياري) يمكن دعم مفتاح عام مثل `all` لتطبيق حقول على جميع الأنواع، إذا تبنّته فرق المنتج/التطبيق.

#### 3.1) تعريف الحقل (Field definition)

- **`type`**: `string` — نوع الخانة، أحد:
  - `text`, `textarea`, `number`, `date`, `email`, `phone`, `url`
  - `select`, `multi_select`, `radio`, `checkbox`, `checkbox_group`
  - `file`
- **`key`**: `string` — مفتاح ثابت بصيغة `snake_case` (مثال: `license_number`)
- **`label`**: `object` — عناوين متعددة اللغات (مثال: `{ "ar": "...", "en": "..." }`)
- **`options`**: `array` — يظهر فقط للأنواع الاختيارية (`select/multi_select/radio/checkbox_group`)
  - `options[].value`: `string`
  - `options[].label`: `object` (i18n)
- **`visibility`**: `object`
  - `mode`: `always | conditional`
  - `when`: يظهر فقط عندما `mode = conditional` (انظر قسم الشروط)
- **`required_when_visible`**: `boolean`
- **`validations`**: `object` (اختياري) — تحقق/قيود حسب نوع الحقل (Regex, min/max, ...)

#### 3.2) `validations` (اختياري) — التحقق/القيود

- **للحقول النصية** (`text`, `textarea`) (اختياري):
  - `min_length`: `number`
  - `max_length`: `number`
  - `pattern`: `string` (Regex) (للاحتياج البسيط/التوافقية)
  - `regex_rules`: `array` (Multiple Regex verifications)
    - `regex_rules[].regex`: `string`
    - `regex_rules[].message`: `object` (i18n) مثل `{ "ar": "...", "en": "..." }`
- **للحقول الرقمية** (`number`) (اختياري):
  - `min`: `number`
  - `max`: `number`
- **للتاريخ** (`date`) (اختياري):
  - `min_date`: `string` (ISO `YYYY-MM-DD`)
  - `max_date`: `string` (ISO `YYYY-MM-DD`)
- **للقوائم المتعددة** (`multi_select`, `checkbox_group`) (اختياري حسب المنتج):
  - `min_selected`: `number`
  - `max_selected`: `number`
- **للملفات** (`file`) (اختياري):
  - `accept`: `array<string>` (مثل: `["pdf","png","jpg"]`) — *يبقى على مستوى الحقل لأنه يصف نوع المحتوى المقبول*
  - `min_size_mb`: `number` (اختياري)
  - `max_size_mb`: `number`

## شروط الظهور (Conditional visibility)

عند `visibility.mode = conditional` يكون `visibility.when` على الشكل:

```json
{ "field": "school_gender", "operator": "equals", "value": "girls" }
```

الحقول:

- **`field`**: `string` — مفتاح الحقل المرجعي الذي يعتمد عليه شرط الظهور
- **`operator`**: `string` — أحد (مقترح):
  - `equals`, `not_equals`
  - `in`, `not_in`
  - `is_true`, `is_false`
  - `exists`
- **`value`**: `any` — قيمة/قيم الشرط (قد تكون `string | number | boolean | array`)

## القيود والتحقق (Validation / Constraints)

- **`key`**:
  - صيغة `snake_case` (يسمح بـ `a-z`, `0-9`, `_`)
  - **فريد** ضمن (الدولة + نوع الكيان)
  - يُفضّل عدم تغييره بعد الاستخدام لتجنب كسر البيانات التاريخية
- **`options`**:
  - إلزامية للأنواع الاختيارية (`select`, `multi_select`, `radio`, `checkbox_group`)
  - كل `value` داخل `options` يجب أن يكون فريدًا ضمن الحقل
- **`visibility.when`**:
  - إلزامي عند الوضع المشروط
  - يجب أن يشير إلى **حقل سابق** في نفس النموذج لتجنب حلقات الاعتماد (حسب سياسة التطبيق)
- **اللغات**:
  - يُفضّل فرض وجود `label` باللغة الأساسية (مثال: `ar`) والسماح ببقية اللغات حسب سياسة الدولة/المنتج

