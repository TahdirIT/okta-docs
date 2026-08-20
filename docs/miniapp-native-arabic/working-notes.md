# ملاحظات العمل — المصائد، نقاط التعديل، الأسئلة المفتوحة

> التطبيقات المصغّرة الأصيلة (Dart على الجهاز). `okta-miniapp@50e394e`،
> `okta-app@e49423b`، `okta-web@52e48ee`، `okta-partners@608abc3`. قُرِئت 2026-08-20.
> ملفات مرافقة: [`README.md`](./README.md) · [`flow.md`](./flow.md) ·
> [`contracts.md`](./contracts.md) · [`data.md`](./data.md) · [`edges.md`](./edges.md)
> · 🇬🇧 [النسخة الإنجليزية](../miniapp-native/working-notes.md)

---

## نتيجة: تثبيت القالب انحرف مجدّداً

**`[مؤكَّد]`، ويستحقّ التصرّف.**

يُولَّد كل مستودع شريك جديد بـ `pubspec.yaml` يثبّت `okta_miniapp` بـ commit ref،
مستبدَلاً من `okta-partners/config/okta-web.php:107`:

```php
'runtime_ref' => env('OKTA_MINIAPP_RUNTIME_REF', '26d0f6cd3ec03ad48fabe43ad4d64e26158ee1b2'),
```

مُتحقَّقٌ منه مقابل تاريخ `okta-miniapp` على هذا الفرع:

| | الـ commit | التاريخ | `oktaHostContractVersion` |
|---|---|---|---|
| الافتراضي المثبَّت | `26d0f6c` | 2026-08-16 | **13** |
| okta-miniapp HEAD | `50e394e` | 2026-08-19 | **17** |

و`26d0f6c` سلفٌ لـ HEAD؛ يفصل بينهما **13 commit**، تمتدّ على العقود 14 و15 و16
و17 — الهوية كقيم مفردة، والصور البعيدة وسحب الكاميرا، والزجاج، وتبويبات ملف
الطالب.

**لماذا يهمّ.** الشريك المولَّد اليوم يترجم مقابل العقد 13. وأي تطبيق مصغّر
يستعمل `Okta.locale()` أو `Okta.remoteImage()` أو `Okta.cameraScanStart()` أو
`okta_glass` أو `Okta.studentHashid()` يفشل في `flutter test tool/validate.dart`
محلياً برمزٍ لم يكتبه الشريك ولا يستطيع إصلاحه. وضمان بوّابة CI — «يُترجَم في CI
يستلزم يُترجَم على الجهاز» — **ينقلب**: صارت ترفض كوداً كان الجهاز سيشغّله.

وهذا هو بالضبط الإخفاق الذي يسجّل `okta-partners/CLAUDE.md` أنه وقع مرّة من قبل
(«بقي القالب على commit سابق للعقد 8 عبر ثلاثة إصدارات عقد»). طُبِّق إصلاحُ
«اجعلها إعداداً لا قيمةً حرفية»؛ ومع ذلك تعفّنت القيمة.

**تحفّظات قبل التصرّف:**
- هذا هو **الافتراضي المُلتزَم في الكود**. وقد يضبط الإنتاج
  `OKTA_MINIAPP_RUNTIME_REF` على قيمة حديثة — ولم يكن أي `.env` مقروءاً هنا.
  `[افتراض]`
- والقيمة الصحيحة ليست HEAD تلقائياً. يجب أن تطابق commit الـ `okta_miniapp`
  الذي بُني عليه **ملف okta-app الثنائي المشحون**، لأن `okta_kit` وأخواتها تعيش
  في ذلك الملف (`contracts.md` §H5). والتثبيت **أمام** التطبيق المشحون هو الخطأ
  نفسه في الاتجاه المعاكس.

**الإصلاح**: سطر واحد في `okta-partners/config/okta-web.php:107`، مع إعادة تشغيل
`pushBoilerplate()` لمستودعات الشركاء القائمة (وهي مهمّة فريق المنصّة، في PR
منفصل).

---

## المصائد — ما يُترجَم ثم يفشل

كل بند أدناه انهيارٌ وصل جهازاً. المصادر: `okta-miniapp/CLAUDE.md`، وREADME
القالب، وترويسات الأدوات.

### المصيدة 1 · الصنف في مكتبة محقونة يجب أن **يُبلَغ** ليوجد

> **الصنف المُعلَن في مكتبة محقونة لا يجوز أن يُذكَر إلا من دالة يستدعيها
> التطبيق المصغّر فعلاً.**

لا يحلّ `dart_eval` مثل هذا الصنف إلا حين يجذبه التطبيق المصغّر. أمّا التطبيق
الذي لا يستدعي الدالة أبداً فالاسم لا يوجد عنده — ويحطّ الفشل **على المكتبة
المحقونة**، مسمّياً ملفاً لم يكتبه الشريك.

سبقت الإجابةَ الصحيحة ثلاثُ تشخيصات خاطئة، وبدت كلٌّ منها مؤكَّدة **لأن المترجِم
يتوقّف عند أول خطأ**، فكان كل إصلاح يكشف الصنف التالي في الطابور: (1) «نوع
الإرجاع المجرّد هو البناء المعطوب» — فإسقاط التعليق نقل الخطأ إلى الجسم؛
(2) «إنه يُترجَم لأنه يشير أيضاً إلى `OktaCapability`» — تخمينٌ يناسب نقطة بيانات
واحدة؛ (3) «الأجسام غير المتزامنة آمنة» — قابلة للفحص، وخاطئة رغم ذلك: فإصلاح
`context()` أنتج فوراً `Unknown type OktaUploadResult` في توقيع **async**.

فصارت `context` و`uploadFile` و`location` و`preciseLocation` تعيد خرائط بسيطة.
`[مؤكَّد]` `okta-miniapp/lib/src/host/okta_host_source.dart:14-34`

**افعل**: اقرأ الخرائط بـ `['status']` / `['latitude']`؛ وفضّل القيم المفردة
`Okta.locale()/tenantId()/roleId()/isDark()`.

### المصيدة 2 · القيمة القادمة من الخارج ليست قيمة مفسِّر

استدعاء دالة مجسَّرة على واحدة منها يموت — و**الرسالة تشير إلى الوسيط لا إلى
المستقبِل**، ولهذا أُسيء قراءتها طويلاً:

```
type 'String' is not a subtype of type '$Value?' in type cast
#0 _CastListBase.[]
#1 $String._startsWith
```

`args[0]` هو الحرف `'ar'`؛ ولا خطأ فيه. **المستقبِل** هو الخاطئ الشكل، فسلك
المترجِم مسار الإرسال الديناميكي ودفع الوسيط غير معلَّب. وأي دالة مجسَّرة تفعلها —
`startsWith`، `toLowerCase`، `trim`، `contains`، `substring`، و`toString()` التي
يسهل نسيان أنها مجسَّرة أصلاً.

مصدران حيّان: قيمة مقروءة من خريطة مفكوكة (`map['locale'] as String` — والـ cast
**ليس** حارساً، فالـ String الآتي من المضيف **هو** String)، وقارئ JSON بنمط
`_text()` يعيد ما وجده.

**أصلِحها عند المنبع، لا عند مواضع الاستدعاء:**

```dart
final dynamic value = map['locale'];
return '$value';          // لا `map['locale'] as String`
```

الاستيفاء يجعل المفسِّر يخصّص النصّ. وللقيم المنطقية استعمل `== true` لا
`as bool`.

وقد اختبأت لأن ثلاثة تطبيقات أعادت اكتشاف حيلة `'$x'` كلٌّ عند مواضع استدعائه،
وكتبها كلٌّ منها كفلكلور محلي. `[مؤكَّد]` `okta_host_source.dart:87-116`

### المصيدة 3 · مفتاح خريطة مفقود ينتج قيمةً لا تستطيع حتى فحصها

جسر `Map` في `dart_eval` يعيد **null خام من Dart** للمفتاح المفقود، لا `$null`
المعلَّب الخاص بزمن التشغيل. و`IsType.run` يبدأ بـ
`runtime.frame[_objectOffset] as $Value`.

فـ `value is String` على مفتاح مفقود لا تعيد false — بل **ترمي**:
`type 'Null' is not a subtype of type '$Value'`. و`== null` أسوأ من عديمة
الفائدة: فـ `CheckEq` يتراجع إلى `==` عادية من Dart، فيكون `rawNull == null`
**false**، فيمرّر الحارس القيمة بصمت إلى النداء التالي، فيموت هو بدلاً عنه.

ولا توجد طريقة دفاعية للنظر إلى مثل هذه القيمة. **لا تُنشئ واحدة أبداً:**

```dart
if (!source.containsKey(key)) return null;   // null على مستوى المصدر هي $null معلَّبة
return source[key];
```

هذا ما تفعله `OktaJson.at` (`okta-miniapp/lib/shared/okta_kit.dart:67`). وعلى أي
قارئ مكتوب يدوياً أن يفعل مثلها. `[مؤكَّد]`

### المصيدة 4 · الوسائط المسمّاة تُطابَق بتساهل وتُنمَّط بصرامة

مقروءة من `helpers/argument_list.dart` في `dart_eval`. يمشي المترجِم على
المعاملات **التي يعلنها الجسر** ويبحث عن كلٍّ منها في ندائك. ثلاث نتائج تعضّ في
اتجاهات متعاكسة:

1. **الوسيط الذي لا يعلنه الجسر لا يُقرأ أبداً** — لا خطأ ولا تحذير، يُهمَل بصمت.
   ويُبقي `flutter_eval` المعاملات غير المدعومة كأسطر `BridgeParameter`
   **معلَّق عليها**، فيبدو `BoxDecoration` كأنه يأخذ `borderRadius` و`gradient`
   و`image` و`shape` — ولا يأخذ **أياً** منها.
2. **`optional: false` غير مفروضة للمعاملات المسمّاة** — فالمعلَن المحذوف يصل
   `null`. وهذه هي الآلية الحقيقية وراء قاعدة `Expanded`/`flex`.
3. **والمعامل المعلَن الذي تمرّره فعلاً يُفحَص نوعياً بصرامة**، مقابل رسم
   `$extends` الخاص بالجسر، وهو لا يعرف شيئاً عن `implements` في Dart:
   فـ `$Border` لا يعلن أي نوع أعلى، فـ `Border` **ليس** `BoxBorder`، و
   `BoxDecoration(border: Border.all(…))` يفشل في الترجمة.

فـ `BoxDecoration` عملياً `color` + `boxShadow`. دوّر الزوايا بـ
`ClipRRect` + `ColoredBox`؛ وارسم الخطّ الشعري بـ `Container(height: 1.0, color: …)`.

### المصيدة 5 · مُعلَن، لكنه غير موصول

قد يُعلَن الصنف للمترجِم ولا يُسجَّل في زمن التشغيل أبداً. فيُترجَم تماماً،
ويجتاز `tool/validate.dart`، ثم يرمي أثناء البناء:

```
UnimplementedError: Tried to invoke a nonexistent external function
```

وفي flutter_eval 0.8.2 هذا صحيح على **`InkWell` و`InkResponse` و`SafeArea`**.
ولا يستطيع فحص الترجمة التقاطه، ولا قراءة قائمة الأصناف المجسَّرة — وهكذا بالضبط
شُحِن شريط تبويبات بـ `InkWell` وانهار على أول جهاز شغّله.

**والمصيدة نفسها موجودة على الدوال، والتدقيق لا يغطّيها.** و`List.sort()` هو
المثال الحيّ: فتنفيذها في زمن التشغيل يقرأ `args[0] as EvalCallable` (لا
`args[0]?`)، فـ `list.sort()` بلا مقارِن يُترجَم ثم يرمي. مرّر المقارِن دائماً،
أو لا تفرز.

**و`State.mounted` غير مجسَّرة** — فالفجوة غير المتزامنة لا تستطيع أن تسأل هل ما
زالت على الشاشة. احرس بعلَمك الخاص.

### المصيدة 6 · أعطال التعليب — أربعة طرق إلى خطأ الـ cast نفسه

كلّها تنتج `type '$X' is not a subtype of type 'X' in type cast`، ويبدو أن موت
موضعٍ بعينه يتوقّف على **تخصيص السجلّات** — فـ«هذا الموضع يعمل» ليس دليلاً.

| الطريق | خطأ | صواب |
|---|---|---|
| بدائية عبر معامل مفسَّر إلى باني مجسَّر | `IconData _icon(int c)` | ابنِها من قيم حرفية عند موضع الإنشاء، أو من دالة بلا وسائط (`OktaIcons`) |
| قراءة **قيمة منطقية من قائمة** | `final dot = dots[i];` | `final dot = i < dots.length && dots[i] == true;` |
| `setState` بـ **جسم سهمي** | `setState(() => _busy = true)` | `setState(() { _busy = true; })` |
| عدّاد حلقة **مُهيّأ من معامل** | `for (int i = from; i <= to; i++)` | حدود حرفية، ودالة لكل مدى |

وآلية الفهرسة في القائمة **مؤكَّدة مقابل مترجِم dart_eval**:
فـ `IndexedReference.getValue` يعيد `Variable.alloc(ctx, listElementType)` وعلَم
`boxed` فيه false لـ `List<bool>`، بينما يسلّم زمن التشغيل `$bool`؛ ثم يُصدِر
`boxIfNeeded` اللاحق `BoxBool` على شيء معلَّب أصلاً. والمقارنة تنجح لأن
`_compileShortCircuit` ينتهي بـ `.copyWith(boxed: true)`، فيعود `boxIfNeeded`
مبكراً.

حوّل **كل** جسم سهمي في `setState`، لا الذي ينهار وحده.

### المصيدة 7 · الشرطي المتداخل يفقد قيمته

```
type 'Null' is not a subtype of type 'String' in type cast
13525: CopyValue (L24 <-- L24)     ← كُتِبت في الخانة 24
13527: BoxInt (L23)  <<< EXCEPTION ← وقُرِئت من الخانة 23، ولم تُكتَب قطّ
```

يخصّص المترجِم خانة نتيجةٍ للشرطي الخارجي، ويكتب الداخلي في خانة أخرى.
**التداخل هو المُطلِق؛ وإلى أين كانت القيمة ذاهبة لا يهمّ** — وسيطاً **أو**
متغيّراً محلياً عادياً. أمّا الشرطي المفرد فسليم.

والإصلاح ليس رفع القيمة إلى متغيّر محلي، لأن المصيدة 6 ما زالت تنطبق على بدائية
تغذّي بانياً مجسَّراً. **تفرّع، وكرّر النداء:**

```dart
if (selected) return OktaMotion.box(-1.0, -1.0, 0xFF6D428F, 999.0, 180, child);
if (_dark)    return OktaMotion.box(-1.0, -1.0, 0xFF1F2A37, 999.0, 180, child);
return              OktaMotion.box(-1.0, -1.0, 0xFFF3F4F6, 999.0, 180, child);
```

أمّا الـ widgets فتُرفَع إلى متغيّر محلي بلا مشكلة — ابنِ الابن مرّة واحدة.
البدائية وحدها مثبَّتة عند موضع النداء.

### المصيدة 8 · المستقبِلات المنمَّطة بـ Map، والـ tear-offs، والحلقات المتداخلة

من README القالب (القائمة الموجَّهة للشريك):

- **افهرس الـ JSON على مستقبِل `dynamic`، لا على واحد منمَّط بـ `Map` أبداً.**
  فحارس `is Map` **يُرقّيه** فيُطلِق المصيدة: يربط dart_eval الـ `Map` الخام
  بتجسيدٍ مجسَّر اعتباطي ويرفض مفتاحك النصّي وقت الترجمة بـ
  `Cannot use variable of type String as index to map of type <int, Color>`.
  و`<int, Color>` لا تأتي من بياناتك أبداً — لا تذهب تبحث عنها. احرس على
  **النتيجة** (`v is List`)، لا على المستقبِل. وترقية `is List` سليمة؛ الـ `Map`
  وحدها المتأثّرة.
- **يجب أن تكون الـ callbacks حرفيات إغلاق**، لا tear-offs:
  `onPressed: () => _f()`، لا `onPressed: _f` («Cannot box Function»).
- **لا `cond ? null : closure`** في خانة callback.
- **لا حلقات متداخلة** — يرمي dart_eval 0.8.5 خطأ `RangeError` وقت الترجمة حين
  يحوي جسم حلقةٍ، مباشرةً **أو عبر دالة يستدعيها**، حلقةً أخرى. سطّح البيانات
  أولاً، ويفضَّل في الخادم.
- **الأزرار: `ElevatedButton` / `TextButton` فقط**؛ بلا معامل `style`، فلا يمكن
  إعطاء الأزرار هوية البراند.
- **`Wrap` و`Divider` و`SingleChildScrollView` و`TextAlign` و`Material` و
  `CircularProgressIndicator` و`Colors` و`TabBar*` غائبة.** ولا بدائيات حركة
  إطلاقاً. و`ThemeData.colorScheme` غائبة، فالتطبيق المصغّر **لا يستطيع** قراءة
  لوحة المضيف — ولهذا يحمل `OktaPalette` قيمه الخاصة.
- **`Text` لا يجسّر `maxLines` ولا `overflow`** — فالعنوان الأعرض من صندوقه
  يلتفّ في منتصف الكلمة. استعمل `FittedBox(fit: BoxFit.scaleDown, …)` داخل عرض
  محدود.
- **كل `Expanded` يحتاج `flex:` صريحاً**؛ و`Flexible` غير مجسَّرة.
- **وثمّة ساعة**: `DateTime` و`Duration` و`Future.delayed` مجسَّرة وموصولة.
  و`Timer` وحدها الغائبة — فلا callback **دوري**، لكن التأخير وطوابع الأحداث
  الحقيقية تعمل. و`dart:convert` (`jsonEncode`/`jsonDecode`) مجسَّرة.

### البوّابات الثلاث، وما لا تلتقطه كلٌّ منها

| البوّابة | تلتقط | عمياء عن |
|---|---|---|
| `flutter analyze` | أخطاء Dart العادية | كل ما يخصّ الصندوق الرملي؛ وأيضاً **تُبلِّغ خطأً أن `Okta`/`package:okta_host` غير معرَّفين** — وهذا متوقَّع |
| `flutter test tool/validate.dart` | أخطاء الترجمة، مقابل زمن التشغيل الحقيقي | الوسائط المُسقَطة بصمت (المصيدة 4.1)، الأصناف غير الموصولة (المصيدة 5)، كل أعطال التعليب (المصيدتان 6–7) |
| `dart run tool/audit_bridge_usage.dart <main.dart>` | أالصنف مجسَّر؟ أالمعامل مقبول؟ أالنوع يفي؟ **أالباني موصول؟** | **الدوال** (`List.sort`)، وأعطال التعليب؛ وهي استدلالية لا مُحلِّل — إيجابيات كاذبة على كلمة مكتوبة بحرف كبير داخل نصّ |

`okta-miniapp/tool/audit_bridge_usage.dart:1-36`. عامِل التدقيق النظيف على أنه
**ضروري لا كافٍ** (`:35`).

---

## نقاط التعديل

### أ · إضافة صلاحية إلى المفردات

مرتَّبة؛ وتخطّي أيٍّ منها ينتج اسماً يُنشَر ولا يُمنَح أبداً.

1. `okta-miniapp/lib/src/host/okta_capabilities.dart` — أضف الثابتة **وأضفها إلى**
   `all` (`:75-89`). *هذه هي النسخة الوحيدة التي تقرأها البوّابة.*
2. `okta-miniapp/lib/src/host/okta_host_delegate.dart` — أضف دالة الـ delegate.
3. `okta-miniapp/lib/src/host/okta_host_plugin.dart` — سجّل دالة الجسر في
   `configureForRuntime` (`:79-141`) **ملفوفةً بـ `_gated(...)`** (`:191-212`)،
   مختاراً قيمة رفض من مفردات `edges.md` §مفردات الرفض. وصلُها هنا هو ما يكشفها؛
   وتبويبها هنا ليس اختيارياً.
4. `okta-miniapp/lib/src/host/okta_host_source.dart` — أضف دالة واجهة `Okta.*`
   **وثابتة اسم** (`:660-702`). والتزم المصيدة 1: لا تسمِّ أي صنف.
5. **ارفع `oktaHostContractVersion`** — `okta_host_delegate.dart:228`.
6. `okta-web/app/Enums/DeviceCapability.php` — أضف حالة الـ enum (`:38-48`).
   لاحظ أن هذا الـ enum يحمل `GATE_CONTRACT` المشترك وحده (`:60`)، بلا حدّ لكل
   حالة.
7. `okta-partners/app/Enums/DeviceCapability.php` — أضف الحالة (`:30-40`) **وحدّها
   الخاص بها** في `minContract()` (`:70-77`). فالحدّ محفوظ لكل حالة هنا، وهنا
   وحدها، «حتى لا تدّعي قدرةٌ أُضيفت في العقد 15 — مثلاً — أنها تعمل على 13» —
   وهو `match` بلا فرع افتراضي، فالحالة الجديدة غير المضافة هناك خطأ PHP لا
   `GATE_CONTRACT` صامتة.
8. `okta-web/lang/{ar,en}/partner_apps.php` — وصف المنصّة الثابت
   (`capability_descriptions`)، الذي لا يستطيع الشريك كتابته.
9. `okta-app/lib/features/miniapps/consent/mini_app_capability_labels.dart` —
   التسمية والوصف المعروضان على النافذة. وهذه **نسخة ثانية من الأسماء**: فالملف
   لا يستطيع استيراد `package:okta_miniapp` (انظر `edges.md` §4)، فالاسم المفقود
   هنا يُبوَّب بشكل صحيح لكنه يسأل بـ**مفتاحه الخام** — وهو ما يبدو خطأً على
   الشاشة الوحيدة التي يقرّر فيها شخصٌ شيئاً.
10. `okta-app/lib/features/miniapps/bridge/okta_host_delegate.dart` — نفّذها،
    بما في ذلك طلب إذن نظام التشغيل.
11. الاختبارات: `okta-miniapp/test/okta_host_contract*_test.dart`،
    و`okta-app/test/mini_app_consent_test.dart` (مجموعة تغطية المفردات في
    `:109-137` تفشل تلقائياً إن تُخطّيت الخطوة 9)،
    و`okta-web/tests/Feature/Store/DeviceCapabilitiesManifestTest.php`،
    و`okta-partners/tests/Feature/Modules/DeviceCapabilitiesManifestTest.php`.
12. اشحن **بناء okta-app جديداً** — فالمكتبات المحقونة تعيش في ذلك الملف الثنائي
    (`contracts.md` §H5). وإعادة نشر التطبيقات المصغّرة لا تفعل شيئاً.
13. حرّك `OKTA_MINIAPP_RUNTIME_REF` (`okta-partners/config/okta-web.php:107`) إلى
    الـ commit الذي استعمله ذلك البناء.

> **الخطوتان 6–7 بلا مترجم بينهما وبين الخطوة 1.** ثلاث نسخ من أحد عشر اسماً،
> ثلاثة مستودعات. والخطوة 9 **مفحوصة آلياً** —
> `okta-app/test/mini_app_consent_test.dart:109-137` يثبت أن التسميات و
> `OktaCapabilities` تغطّي إحداهما الأخرى بالضبط، في الاتجاهين. **أمّا enum الـ
> PHP فلا.** لدى `okta-partners` اختبار يثبت أن enum لديه يطابق مخطّط الـ manifest
> والقالب، لكن كما يقول `CLAUDE.md` الخاص به:
> *«نجاحه ليس دليلاً على أن الجهاز يعرف الاسم»*. والاسم الموجود في enum الـ PHP
> وغير الموجود في `OktaCapabilities` يُنشَر نظيفاً ثم لا يُمنَح أبداً.

### ب · إضافة رمز إلى مكتبة محقونة (مثل دالة على `OktaTabs`)

1. عدّل `okta-miniapp/lib/shared/okta_kit.dart`. والتزم **قاعدة الاكتفاء الذاتي**
   — بلا أي إشارة إلى صنف آخر في المكتبة نفسها.
2. `dart run tool/generate_kit_source.dart` و**التزم الـ `.g.dart` في git**.
   و`test/okta_kit_test.dart` يُرسِب البناء عند الانحراف.
3. أضف حالة ترجمة إلى `test/okta_kit_test.dart` — فهو يترجم تطبيقاً مصغّراً
   مقابل كل نقطة دخول عامة تحديداً لأن المحلّل أعمى هنا.
4. **ارفع `oktaHostContractVersion`.** فالدالة الجديدة كاسِرةٌ كالمكتبة الجديدة؛
   وهذه هي حادثة `OktaTabs.bottomBar` (`contracts.md` §H5).
5. الخطوتان 12–13 من §أ.

يتحرّك `oktaMiniAppRuntimeSignature` من تلقاء نفسه (فهو يبصم المصادر) وسيُبطِل
الكاش بشكل صحيح — لكنه **لا يبوّب القبول**. الرفع وحده يفعل.

### ج · إضافة مكتبة محقونة جديدة

سجّلها في `okta-miniapp/lib/src/injected_sources.dart:41-46` **وفي لا مكان
آخر** — فتلك الخريطة الواحدة هي ما يحقنه الـ plugin وما يبصمه مفتاح الكاش معاً،
فلا يستطيع الشحنُ وإبطالُ الكاش أن ينحرفا. ثم ارفع العقد، ثم الخطوتان 12–13 من §أ.

### د · تغيير شكل مسار الدخول الأصيل

ثلاثة مواضع، مستودعان (انظر `contracts.md` §H3): `MobileEntryRules.php:36`،
و`ManifestValidator.php:818`، و`BundleMiniappSource.php:50`. والثالث يخدم أيضاً
مودولات مخزَّنة قبل وجود بوّابة النشر، فلا يستطيع أن يفوّض ببساطة.

### هـ · شحن بناء okta-app جديد

دائماً `--no-tree-shake-icons`، على **كل** هدف إصدار (`data.md` §الأعلام).

---

## الأسئلة المفتوحة

كلٌّ منها يجيب عنه بجملة تقريباً من يعرف أصلاً.

1. **هل `OKTA_MINIAPP_RUNTIME_REF` مضبوط في الإنتاج**، وعلى ماذا؟ الافتراضي
   المُلتزَم يثبّت العقد 13 مقابل HEAD على 17. (§نتيجة)
2. **أي commit من `okta_miniapp` بُني عليه okta-app المشحون حالياً؟** ذلك، لا
   HEAD، هو التثبيت الصحيح للقالب.
3. **هل يوجد أي فحص آلي أن enum الـ `DeviceCapability` في الـ PHP يوافق
   `OktaCapabilities`؟** النصف الخاص بـ okta-app ↔ okta-miniapp **مغطّى**:
   فـ `okta-app/test/mini_app_consent_test.dart:109-137` يثبت في الاتجاهين أن كل
   اسم في `OktaCapabilities.all` له تسمية، وأن لا تسمية تسمّي ما ترفضه
   `OktaCapabilities.isKnown`. ولم يُعثَر على ما يعادله بين مفردات Dart و enum
   الـ PHP. (§أ الخطوتان 6–7)
4. **`okta-miniapp/tool/validate.dart:11` يجعل الافتراضي لـ `OKTA_MINIAPP_DIR` هو
   `miniapp_dart`** — الاسم السابق لـ `okta_app/native/`. أهو افتراضٌ ميّت، أم ما
   زال يستعمله شيء؟ نسخة القالب نفسها تثبّت `lib/` في الكود، فالشركاء غير
   متأثّرين. `[مؤكَّد]` كانحراف، ومجهول الأثر.
5. ~~**هل كتابات الكاش ذرّية أمام قتلٍ في منتصف الكتابة؟**~~ **أُجيب، وأُصلِح.**
   لم تكن ذرّية: `writeAsBytes` كان يكتب مباشرةً على الهدف، فالقتل أثناء الكتابة
   يترك `.evc` مبتوراً تعيده `read()` بكل رضا ثم يموت عليه `Runtime(ByteData)`
   في كل إقلاع حتى ينشر الشريك إصداراً جديداً. صار `write()` يكتب على
   `<target>.writing` ثم يعيد التسمية — وإعادة التسمية إمّا وقعت أو لم تقع.
   وفي الجولة نفسها أُصلِحت `read()` التي كانت تُعيد الـ future من داخل `try`
   (`return f` بدل `return await f`)، فكان خطأ الإدخال/الإخراج يهرب من الدالة
   التي وعد توثيقها بأنها لا ترمي أبداً.
6. **هل يمكن أن يُترجَم تطبيقان مصغّران تزامناً**، وهل الترجمة على الجهاز مقيَّدة
   بالمعالج بما يكفي ليهمّ على الأجهزة الضعيفة؟ لم يُتتبَّع.
7. **هل يعيد أي شيء تشغيل `pushBoilerplate()` لمستودعات الشركاء القائمة**، أم أن
   كل مستودع مولَّد مجمَّدٌ على تثبيت لحظة إنشائه؟ يسمّيها
   `okta-partners/CLAUDE.md` مهمّة فريق المنصّة؛ ولم يُعثَر على أي مجدوِل.
8. **`microphone` محجوزة بلا رمز جسر.** أثمّة قدرة قادمة، أم ينبغي إزالتها من
   المفردات؟ الشريك الذي يعلنها اليوم يستدعي تحذير نشر «مُعلَنة وغير مستعملة».
9. **هل يرفض `toolKeyStart()` فعلاً فقط حين لا تكون أيٌّ من قنواته الأربع
   ممنوحة، والمضيف يرشّح قناةً قناة؟** مذكور في `okta-miniapp/CLAUDE.md` §العقد 13
   ومتّسق مع قائمة غير المبوَّبة، لكن الترشيح لكل قناة نفسه يعيش في
   `okta-app/lib/features/toolkeys/` ولم يُتتبَّع.
   (`edges.md` §مفاتيح الأدوات تُبوَّب خشِناً، موسومة `[مرجَّح]`)
10. **هل تعمل الاستيرادات النسبية داخل حزمة تطبيق مصغّر؟** كل ملفات `lib/`
    تُسلَّم كحزمة واحدة، والمثال الوحيد متعدّد الملفات
    (`okta-miniapp/test/okta_mini_app_bundle_test.dart:12-30`) يستعمل `package:`
    حصراً. ولم يكن Flutter SDK متاحاً هنا لتجربة `import 'widgets/card.dart'`.
    إن كانت تعمل فهي تستحقّ التوثيق؛ وإن لم تكن، فتستحقّ قاعدةً صريحة في README
    القالب. `[افتراض]` (`data.md` §تعدّد الملفات وصيغة الاستيراد)

---

## فهرس المراسي

الملفات الجديرة بالفتح أولاً، حسب السؤال.

| السؤال | الملف |
|---|---|
| ماذا يستطيع التطبيق المصغّر أن ينادي؟ | `okta-miniapp/lib/src/host/okta_host_source.dart` |
| ماذا ينفّذ المضيف؟ | `okta-miniapp/lib/src/host/okta_host_delegate.dart` (العقد في `:228`) |
| كيف يُوصَل ويُبوَّب؟ | `okta-miniapp/lib/src/host/okta_host_plugin.dart` (`:79`، `:191`) |
| ما الذي يُحقَن؟ | `okta-miniapp/lib/src/injected_sources.dart` |
| مفتاح الكاش | `okta-miniapp/lib/src/cache/okta_mini_app_file_cache.dart` (`:57-71`) |
| بصمة زمن التشغيل | `okta-miniapp/lib/src/runtime_info.dart` |
| أسماء الصلاحيات | `okta-miniapp/lib/src/host/okta_capabilities.dart` |
| الـ widgets/المنطق المشترك | `okta-miniapp/lib/shared/okta_kit.dart` |
| صيغة السلك | `okta-miniapp/lib/src/bundle/okta_mini_app_bundle.dart` (`:176-202`) |
| بوّابة CI | `okta-miniapp/lib/src/validation/okta_mini_app_validator.dart` |
| الجلب/الترجمة/الرسم على الجهاز | `okta-app/lib/features/miniapps/bridge/okta_dart_miniapp_host.dart` |
| delegate الجهاز + الموافقة | `okta-app/lib/features/miniapps/bridge/okta_host_delegate.dart` (`:626`، `:647`) |
| تخزين الموافقة | `okta-app/lib/features/miniapps/consent/mini_app_consent_store.dart` |
| حدّ الاستيراد | `okta-app/.github/workflows/miniapp-boundary.yml` |
| تجميع الحزمة | `okta-web/app/Services/MobileAppCatalog/BundleMiniappSource.php` |
| خدمة الحمولة | `okta-web/app/Http/Controllers/App/MiniappPayloadController.php` |
| الإطلاق | `okta-web/app/Http/Controllers/Api/Mobile/MobileAppCatalogController.php` (`:104`، `:150`) |
| قواعد النشر | `okta-web/app/Modules/Core/ManifestValidator.php` (`:446`، `:817`، `:912`) |
| شكل مسار الدخول | `okta-partners/app/Services/Partners/Manifest/MobileEntryRules.php` |
| القالب | `okta-partners/resources/partner-boilerplate/okta_app/native/main/` |
| تثبيت القالب | `okta-partners/config/okta-web.php:107` |
| **المرجع النثري** | `okta-miniapp/CLAUDE.md` — أعمق سرد مكتوب للمصائد |
