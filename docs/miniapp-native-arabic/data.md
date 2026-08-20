# البيانات والمفردات والإعدادات

> التطبيقات المصغّرة الأصيلة (Dart على الجهاز). `okta-miniapp@50e394e`،
> `okta-app@e49423b`، `okta-web@52e48ee`، `okta-partners@608abc3`. قُرِئت 2026-08-20.
> ملفات مرافقة: [`README.md`](./README.md) · [`flow.md`](./flow.md) ·
> [`contracts.md`](./contracts.md) · [`edges.md`](./edges.md) · [`working-notes.md`](./working-notes.md)
> · 🇬🇧 [النسخة الإنجليزية](../miniapp-native/data.md)

لا شيء على المسار الأصيل مخزَّن في جدول قاعدة بيانات خاصّ بالتطبيقات المصغّرة.
البيانات هي: مصدرٌ على قرص الشريك، وعمودُ manifest، وحمولةُ JSON في الطريق،
و bytecode في كاش الجهاز، وملفُّ موافقة. ويملك كلَّ واحد منها **مستودع واحد
بالضبط**.

---

## أين يعيش كل جزء

| الجزء | المالك | الموضع | العمر |
|---|---|---|---|
| مصدر Dart | مستودع الشريك | `okta_app/native/<entry>/lib/**.dart` | git |
| إعلان السطح | okta-partners | `mobile_config` (JSON) على `partner_module_versions`، و`mobile_entry` لكل جمهور | لكل إصدار |
| الـ manifest المثبَّت | okta-web | صفّ `modules`، يُقرأ عبر `Module::decodedManifest()` | لكل تثبيت، يُعاد كتابته كل تثبيت |
| حزمة المصدر | — | في الطريق فقط؛ `no-store, private` | طلب واحد |
| الـ bytecode المترجَم | okta-app | مجلد دعم التطبيق، `okta_miniapp_cache/` | حتى يتحرّك الإصدار أو بصمة زمن التشغيل |
| قرارات الموافقة | okta-app | ملف JSON لكل (مستخدم، slug) | حتى إلغاء تثبيت التطبيق |

**صلاحيات الجهاز تسافر في الحمولة، لا على بطاقة الكتالوج.** فالمستهلك هو بوّابة
الصلاحيات في زمن التشغيل، وهي لا توجد إلا بعد أن تعمل الحمولة؛ أمّا البطاقة
فتقرّر فقط أتُرسَم مربّعة أم لا. `[مؤكَّد]`
`okta-web/app/Services/MobileAppCatalog/BundleMiniappSource.php:38-45`،
`okta-web/app/Http/Controllers/App/MiniappPayloadController.php:33-36`

## الحزمة (bundle)

`okta-miniapp/lib/src/bundle/okta_mini_app_bundle.dart:65`

| الحقل | مفتاح السلك | الافتراضي | المصدر |
|---|---|---|---|
| `slug` | `slug` | — | **يُحقَن من جهة العميل من الإطلاق** |
| `package` | `package` | `mini_app` | `name:` من `pubspec.yaml` للشريك |
| `files` | `files` | — | كل `.dart` تحت `lib/` للمدخل، مفهرسةً نسبةً إلى `lib/` |
| `entryFile` | `entry_file` | `main.dart` | مدخل الـ manifest، منزوعاً منه بادئة `okta_app/native/<entry>/lib/` |
| `entryFunction` | `entry_function` | `main` | مثبَّت في الخادم |
| `minContract` | `min_contract` | `1` | الـ manifest |
| `capabilities` | `capabilities` | `[]` | الـ manifest، مطبَّعةً مرّتين |
| `payloadVersion` | `payload_version` | — | **يُحقَن من جهة العميل من الإطلاق** |

الحدود المفروضة عند التجميع — `BundleMiniappSource.php:32-36`:
**256 كيلوبايت** للملف، **2 ميغابايت** للحزمة، **128** ملفاً.

### تعدّد الملفات وصيغة الاستيراد

تُجمَع شجرة `lib/` كاملةً تكرارياً (`BundleMiniappSource.php:80-82`) وتُسلَّم
للمترجِم كحزمة Dart **واحدة** (`packages: {package: files}` —
`okta_mini_app_bundle.dart:170-174`). فاستورد ملفاتك بعضها بعضاً بـ `package:`
واسم حزمتك من `pubspec.yaml`:

```dart
// lib/main.dart          (package: 'demo_app')
import 'package:demo_app/widget.dart';          // ملف بجانبه
import 'package:demo_app/widgets/card.dart';    // lib/widgets/card.dart
```

مثال حيّ مُختبَر في `okta-miniapp/test/okta_mini_app_bundle_test.dart:12-30`.
والمجلدات المتداخلة تعمل: المسار النسبي إلى `lib/` يُحفَظ كمفتاح
(`BundleMiniappSource.php:99`).

**الاستيرادات النسبية** (`import 'widgets/card.dart'`) لم تُختبَر في هذا
المستودع — التزم `package:`. `[افتراض]`

وملف الدخول لا يلزم أن يكون `main.dart`: الـ regex يسمح بمسار متداخل، فـ
`…/lib/src/app.dart` مدخل صالح. الملزَم أن تكون **الدالة** اسمها `main`، فـ
`entry_function` مثبَّت في الخادم (`BundleMiniappSource.php:116`).

## المكتبات المحقونة الأربع

`okta-miniapp/lib/src/injected_sources.dart:41-46`. هذه الخريطة هي التعريف
**الوحيد** للسطح المحقون: `OktaHostPlugin` يسجّل هذه بالضبط، و
`oktaMiniAppRuntimeSignature` يبصم هذه بالضبط. قائمتان قد تختلفان؛ قائمة واحدة
لا تستطيع. `[مؤكَّد]` `:35-39`

| الاستيراد | النوع | لماذا هو من هذا النوع |
|---|---|---|
| `package:okta_kit/okta_kit.dart` | **مصدر** | Dart عادية. تبقى ملفاً حقيقياً (`lib/shared/okta_kit.dart`، 1567 سطراً) ليفحصها المحلّل وليستوردها المضيف أصيلةً. تُنسخ حرفياً إلى `lib/src/shared/okta_kit_source.g.dart` عبر `tool/generate_kit_source.dart`؛ و`test/okta_kit_test.dart` يُرسِب البناء عند الانحراف. |
| `package:okta_host/okta_host.dart` | **واجهة جسر** | تنادي `$hostApiCall` وأخواتها — رموزٌ لا توجد إلا كتصريحات جسر داخل الصندوق الرملي. فلا يمكن أن تكون ملفاً حقيقياً يترجمه المضيف، ولذلك تعيش كنصّ اضطراراً. |
| `package:okta_motion/okta_motion.dart` | **واجهة جسر** | الصندوق الرملي **لا يملك أي بدائية حركة إطلاقاً** — لا `Tween` ولا `Curves` ولا `TickerProvider`. فالحركة تعمل أصيلةً، والتطبيق المصغّر يصف الحالة الهدف فقط. |
| `package:okta_glass/okta_glass.dart` | **واجهة جسر** | لا `BackdropFilter` ولا `ImageFilter`؛ و`BoxDecoration` يُسقط `gradient` بصمت ويرفض `Border` صراحةً. فيطلب التطبيق المصغّر من المضيف اللوح الجاهز — وهذا أيضاً ما يتيح لأوكتا إعادة تنسيق المادّة دون إعادة بناء أي تطبيق منشور. |

**«مصدر» مقابل «جسر» حدٌّ أمني لا خيار أسلوبي:**

| | مصدر (`addSource`) | جسر (`registerBridgeFunc`) |
|---|---|---|
| يعمل | مفسَّراً، داخل الصندوق | أصيلاً، في المضيف |
| الامتيازات | **لا شيء** | **كلّها** — تحرسها أنت |
| القيد | يجب أن يبقى داخل مجموعة dart_eval الفرعية | أي Dart |
| لأجل | النماذج، المنطق الصِرف، التنسيق، الـ widgets العرضية | التوكنات، الشبكة، الكاميرا، التنقّل |

**الافتراضي هو المصدر.** دالة الجسر باب جديد للخروج من الصندوق تحرسه أنت وحدك.

### نقاط الدخول العامة في `okta_kit`

`okta-miniapp/lib/shared/okta_kit.dart` — `OktaJson` (`:59`)، `OktaText`
(`:188`)، `OktaPalette` (`:239`)، `OktaAppBar` (`:357`)، `OktaIcons` (`:619`)،
`OktaTabs` (`:690`)، `OktaMetrics` (`:1050`).

**كل صنف هنا قائم بذاته، وهذا ليس تفضيلاً أسلوبياً** — فالصنف في مكتبة محقونة
لا يجوز أن يشير إلى *أي صنف آخر في المكتبة نفسها*، ولذلك يحمل `OktaList` نسخته
الخاصة من مربّع الأيقونة الذي يملك `OktaSurface` مثله. والمحلّل **لا يرى ذلك**:
فالإشارة أصيلةً Dart صحيحة. ويترجم `test/okta_kit_test.dart` تطبيقاً مصغّراً
مقابل **كل** نقطة دخول عامة لهذا السبب بالضبط. `[مؤكَّد]`
`okta-miniapp/CLAUDE.md` §"One class in an injected library cannot see another"

## واجهة `Okta` — كامل سطح القدرات

`okta-miniapp/lib/src/host/okta_host_source.dart:74`. مجمَّعةً حسب العقد الذي
أدخلها؛ ورقم السطر هو سطر التصريح.

| المجموعة | الأعضاء |
|---|---|
| الهوية (قيم مفردة — **فضّلها**) | `locale():117` `tenantId():123` `roleId():129` `isDark():135` `studentHashid():151` `portal():165` |
| الهوية (خريطة) | `context():203` — تعيد **خريطة مفكوكة**، تُقرأ بـ `['locale']` |
| العقد | `contract():76` |
| الـ API | `api():214` `get():233` `getQuery():237` `post():242` |
| المسح | `scanBarcode():247` `scanNfc():250` |
| الملفات | `uploadFile():257` `openDocument():411` |
| الواجهة | `toast():266` `appIcon():354` `remoteImage():569` `cameraPreview():649` |
| التخزين (ع5) | `storeGet():274` `storePut():279` `storeDelete():284` `storeKeys():291` |
| الصوت (ع5) | `playSound():306` |
| الموقع (ع5/ع6) | `location():318` `preciseLocation():338` |
| التنقّل (ع9) | `close():375` |
| مفاتيح الأدوات (ع12) | `toolKeyStart():430` `toolKeyStop():439` `toolKeyReads():474` `toolKeyChannels():488` |
| الصلاحيات (ع13) | `hasCapability():517` `requestCapability():534` + إحدى عشرة ثابتة اسم `:660-702` |
| سحب الكاميرا (ع15) | `cameraScanStart():600` `cameraScanStop():610` `cameraScanReads():631` |

`context` و`uploadFile` و`location` و`preciseLocation` كلّها تعيد **خرائط مفكوكة
بسيطة** ولا تسمّي أي صنف. و`OktaApiResponse` هو الصنف الوحيد الذي ما زال
مذكوراً، وهذا **رِهانٌ مقصود لا برهان**: كل تطبيق مصغّر يصل بياناته عبر
`Okta.get`، فـ `api()` مُستدعاة دائماً. `[مؤكَّد]`
`okta_host_source.dart:14-34`. والقاعدة وراء ذلك في `working-notes.md` §المصيدة 1.

## مفردات الصلاحيات — ثلاث نسخ، بلا مترجم بينها

أحد عشر اسماً، ويجب أن تتطابق عبر ثلاثة مستودعات:

| المستودع | الملف | الشكل |
|---|---|---|
| **okta-miniapp** | `lib/src/host/okta_capabilities.dart` | ثوابت نصية في `OktaCapabilities`، و`all:75`، و`isKnown():94` |
| okta-web | `app/Enums/DeviceCapability.php:38-48` | enum مدعوم |
| okta-partners | `app/Enums/DeviceCapability.php:30-40` | enum مدعوم |

```
camera  microphone  nfc  bluetooth  usb  local_network
location_coarse  location_precise  files_read  documents_open  storage
```

**نسخة okta-miniapp هي الوحيدة التي تُفحَص فعلاً.** البوّابة على الجهاز تقرأها؛
والنسختان الأخريان تقرّران فقط ما **يجوز إعلانه**. فالاسم الموجود في enum الـ PHP
وغير الموجود في `OktaCapabilities` يُنشَر نظيفاً ثم **لا يُمنَح أبداً** — البوّابة
تفشل مغلقة. `[مؤكَّد]`
`okta-miniapp/lib/src/host/okta_capabilities.dart:94`،
`okta-partners/CLAUDE.md` §`mobile.capabilities`

و`GATE_CONTRACT = 13` معلَن في كلا enum الـ PHP (`okta-web:60`،
`okta-partners:50`)، وهو أدنى `minContract` لإعلان أي صلاحية.

أمّا `microphone` فهي **محجوزة**: قابلة للإعلان والمراجعة والموافقة، لكن لا رمز
جسر خلفها. `[مؤكَّد]` `okta_capabilities.dart:22-31`

والمجموعة **غير المبوَّبة** مكتوبة صراحةً، ولكلٍّ سببها، في
`oktaUngatedCapabilities` (`okta_capabilities.dart:133-145`) — لأن «نسينا أن
نبوّبها» و«قرّرنا أنها لا تحتاج بوّابة» يبدوان متطابقين في الكود بعد ستة أشهر.

## مفتاح الكاش

`okta-miniapp/lib/src/cache/okta_mini_app_file_cache.dart`

```
<baseDir>/<slug>__<entry>/<payloadVersion>__<runtimeSignature>.evc
```

المجلد: `_slugDir():64-66`. الملف: `_file():68-71`.

أربعة مُدخَلات، كلٌّ منها يجيب سؤالاً مختلفاً:

| المُدخَل | يتحرّك حين | بدونه |
|---|---|---|
| `slug` | تطبيق مختلف | تتصادم التطبيقات |
| `entry` | حزمة مختلفة تحت slug واحد | **تتصادم حزمتان لـ slug واحد** — تتشاركان الـ slug و`payloadVersion` والبصمة، فمن أُطلِق أولاً كسب مدخل الكاش وأعاد الآخر تشغيل **bytecode الأول** (`:8-21`) |
| `payloadVersion` | يتحرّك `updated_at` لصفّ المودول — **أي تثبيت يحمل commit جديداً**، لا النشر وحده | إصدار جديد يعيد تشغيل bytecode قديم |
| `runtimeSignature` | يتغيّر أي مصدر محقون | إصلاح في المضيف لا يصل الأجهزة أبداً (`:27-32`) |

و`write()` يقلّم كل ملف آخر في المجلد — وهذا ما ينظّف `payloadVersion` القديم،
وهو سبب كون المجلد لكل `(slug, entry)` لا لكل slug (`:57-62`).

### `oktaMiniAppRuntimeSignature`

`okta-miniapp/lib/src/runtime_info.dart:28` =
`oktaMiniAppToolchainSignature` (`:8`، حالياً
`dart_eval-0.8.5+flutter_eval-0.8.2`) + `inj-<digest of every injected source>`.

والبصمة ليست زينة. فـ `payloadVersion` هو إصدار **الشريك** ولا يتحرّك أبداً حين
يتغيّر المضيف، فبصمةٌ تتجاهل المحتوى كانت ستترك الأجهزة تشغّل bytecode مترجَماً
مقابل مكتبات محقونة قديمة — **بصمت**، حتى ينشر كل شريك مصادفةً. `[مؤكَّد]` `:13-21`

وهي **ليست تعمويةً** عمداً: فهذا مفتاح كاش لا حدّ ثقة، والتجزئتان النمطيتان
تبقيان تحت 2^53 في كل خطوة فتتفق النتيجة على الويب وعلى native (`:22-27`).

## الإعدادات والأعلام

| الإعداد | الموضع | القيمة المقروءة | ملاحظة |
|---|---|---|---|
| `OKTA_MINIAPP_RUNTIME_REF` | `okta-partners/config/okta-web.php:107` | الافتراضي المُلتزَم `26d0f6c…` | يُستبدَل كـ `__MINIAPP_REF__` في كل مستودع شريك مولَّد. **قديم — انظر `working-notes.md`.** |
| حدّ Flutter الأعلى | `okta-miniapp/pubspec.yaml` | `>=3.19.0 <3.39.0` | حامل، لا ترتيب — انظر أدناه |
| `--no-tree-shake-icons` | كل بناء إصدار من okta-app | مطلوب | `okta-app/BUILDING.md:65-71` |
| حدّ `mobile.min-version` | صفحة إعدادات كتالوج تطبيقات الجوال في okta-web | — | 426 للعملاء القدامى، `routes/api.php:113-115` |

### حدّ Flutter الأعلى

يجسّر `flutter_eval` الـ SDK بكتابة `class $X implements X` يدوياً لـ 185 صنفاً
من Flutter، مُمرِّراً كل عضو. و`implements` يطالب بالواجهة كاملةً، فأي عضو
**تضيفه** Flutter لاحقاً هو خطأ ترجمة في حزمة لم يعدّلها أحد
(`Container.isAntiAlias` هو المثال الحيّ). ولا يوجد ما يُرقّى إليه: 0.8.2 هي
الأحدث على pub.dev. والحدّ يحوّل «مئات الأخطاء في عمق أرشيف iOS» إلى سطر واحد من
`pub get`. `[مؤكَّد]` `okta-miniapp/pubspec.yaml:8-27`

وهو مكرَّر حرفياً في `okta-app/pubspec.yaml` وفي كل `pubspec.yaml` لتطبيق مصغّر،
لأن `okta_miniapp` يُستهلَك كاعتماد git **مثبَّت بـ commit ref** — فالقيد المضاف
في مكان واحد لا يصل تلك الحلول حتى يحرّك أحدٌ الـ ref.

### `--no-tree-shake-icons`

مطلوب على **كل** هدف إصدار — apk، appbundle، ipa، web. فـ Flutter يجزّئ خطّ
الأيقونات بمسح `IconData` الثابتة وقت البناء، ولا يستطيع النظر داخل كود يُترجَم
على الجهاز، فيقتطع التجزيء بالضبط المحارف التي تطلبها التطبيقات المصغّرة فتُرسَم
مربّعات فارغة — **بصمت**.

وستقابل النصف الصاخب أولاً: `flutter_eval` يبني `IconData` غير ثابتة في مصدره،
فيوقع باحث الثوابت و**يُرسِب البناء**. أبقِ العلَم من أجل النصف الصامت بصرف النظر.
`[مؤكَّد]`
`okta-app/lib/features/miniapps/bridge/okta_dart_miniapp_host.dart:8-19`،
`okta-app/BUILDING.md:65-81`
