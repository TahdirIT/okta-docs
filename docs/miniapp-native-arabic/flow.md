# المسار الذهبي — من ضغطة مفتاح الشريك إلى الـ widget المرسوم

> التطبيقات المصغّرة الأصيلة (Dart على الجهاز). `okta-miniapp@50e394e`،
> `okta-app@e49423b`، `okta-web@52e48ee`، `okta-partners@608abc3`. قُرِئت 2026-08-20.
> ملفات مرافقة: [`README.md`](./README.md) · [`contracts.md`](./contracts.md) ·
> [`data.md`](./data.md) · [`edges.md`](./edges.md) · [`working-notes.md`](./working-notes.md)
> · 🇬🇧 [النسخة الإنجليزية](../miniapp-native/flow.md)

المرساة: **يكتب الشريك `Widget main()` في ملف `.dart`، فيُرسَم كبطاقة داخل
`okta-app`.** كل قفزة أدناه على ذلك المسار وحده. التفرّعات مجموعة في
§التفرّعات الجانبية بدل تتبّعها.

> **المرساة حالةٌ تُتتبَّع، لا قيدٌ على البنية.** التطبيق المصغّر ليس ملفاً
> واحداً: تُجمَع **شجرة `lib/` كاملةً** في الحزمة (128 ملفاً كحدّ أقصى، انظر
> القفزة 7)، وتستورد ملفاتك بعضها بعضاً بـ `package:<اسم_حزمتك>/…`. و
> `entryFile` يسمّي فقط أي ملف يحوي `main`.

---

## القفزة 0 — تجهيز مستودع الشريك

يُنشئ `okta-partners` مستودعاً جديداً للشريك وينسخ إليه هيكلاً حرفياً، بما في
ذلك المجلدات المخفية.

- `okta-partners/app/Services/GitHubAppService.php` ← `pushBoilerplate()` ينسخ
  `resources/partner-boilerplate/**`.
- الحزمة الأصيلة تحطّ في
  `resources/partner-boilerplate/okta_app/native/main/` — `lib/main.dart`،
  `pubspec.yaml`، `analysis_options.yaml`، `tool/validate.dart`، `README.md`.
- تُستبدَل عناصر القالب في
  `okta-partners/app/Services/GitHubAppService.php:386`، حيث
  `__MINIAPP_REF__` ← `config('okta-web.miniapp.runtime_ref')`
  (`okta-partners/config/okta-web.php:107`، متغيّر البيئة `OKTA_MINIAPP_RUNTIME_REF`).

ملف `pubspec.yaml` المولَّد يثبّت `okta_miniapp` **بـ commit ref**، لا بمدى
إصدارات — عمداً، لأن على الشريك أن يترجم مقابل زمن التشغيل نفسه الذي يحمله
ملف `okta-app` الثنائي المشحون. `[مؤكَّد]`
`okta-partners/resources/partner-boilerplate/okta_app/native/main/pubspec.yaml:18-22`

> ⚠️ هذا التثبيت قديم حالياً. انظر `working-notes.md` §نتيجة.

## القفزة 1 — الشريك يكتب Dart

`okta_app/native/<entry>/lib/main.dart`، ويُصدِّر **دالة `Widget main()` عليا**.
المقطع `<entry>` يسمّي حزمة Dart قائمة بذاتها داخل مستودع الشريك؛ وقد يشحن
الـ slug الواحد عدّة حزم. `[مؤكَّد]`
`okta-web/app/Services/MobileAppCatalog/BundleMiniappSource.php:20-27`

الباب الوحيد للخروج من الصندوق الرملي هو واجهة `Okta.*` الساكنة من
`package:okta_host`. وتلك الحزمة **ليست حزمة pub حقيقية** — يحقنها زمن التشغيل
كمصدر وقت الترجمة، ولهذا يُبلِّغ `flutter analyze` أنها غير معرَّفة، ولهذا فإن
ذلك **متوقَّع** لا خطأ في الإعداد. `[مؤكَّد]`
`okta-partners/resources/partner-boilerplate/okta_app/native/main/README.md`
(قسم “Develop & validate”)

## القفزة 2 — الشريك يتحقّق محلياً (بوّابة CI)

```bash
cd okta_app/native/main
flutter pub get
flutter test tool/validate.dart
```

يقرأ `tool/validate.dart` كل ملف `.dart` تحت `lib/` في `OktaMiniAppBundle` ثم
يشغّل `OktaMiniAppValidator().validate(bundle)`. `[مؤكَّد]`
`okta-partners/resources/partner-boilerplate/okta_app/native/main/tool/validate.dart:44-58`

ويبني المدقّق **المترجِم نفسه** الذي يبنيه الجهاز —
`flutterEvalPlugin` + `OktaHostPlugin.compileOnly()` — ويبلّغ بالنجاح أو بنصّ
خطأ المترجِم نفسه، دون إنتاج أي ملف. `[مؤكَّد]`
`okta-miniapp/lib/src/validation/okta_mini_app_validator.dart:44-50`

**وهذا هو الضمان الحامل لكل المسار الأصيل:** لأن الـ commit نفسه من
`okta_miniapp` هو من يترجم في CI ويترجم على الجهاز، فإن «يُترجَم في CI» يستلزم
حتماً «يُترجَم على الجهاز». `[مؤكَّد]`
`okta-miniapp/lib/src/validation/okta_mini_app_validator.dart:31-36`

لكنها بوّابة **ترجمة** فقط. ثلاثة أصناف كاملة من الإخفاق تمرّ منها نظيفةً —
المعاملات المُسقَطة بصمت، والأصناف المعلَنة غير الموصولة، وأعطال التعليب التي لا
تنفجر إلا وقت التشغيل. انظر `working-notes.md` §المصائد.

## القفزة 3 — الشريك يُعلن السطح

على `okta-partners`، **لكل إصدار** (لا توجد قيمة على مستوى المودول)، يضبط
الشريك: `mobile_support` مفعّلاً، `mode: native`، `runtime: dart`، مسار الدخول
لكل جمهور، `min_contract`، وأي صلاحيات جهاز مع تبرير مكتوب لكلٍّ منها.

ثلاثة مسارات كتابة تصل الحقول نفسها، ويجب أن تبقى متّسقة — محرّر الإصدار في
Livewire، ومستورد الـ manifest، وأداة MCP المستضافة:

- `okta-partners/app/Livewire/Partner/Modules/VersionEditor.php`
- `okta-partners/app/Services/Partners/Manifest/PersistImportedSurfaces.php`
- `okta-partners/app/Services/PartnerMcp/Server/Tools/ControlTools.php:114`
  (`set_mobile_surface`) وما بعد `:1404`.

شكل مسار الدخول معرَّف مرّة واحدة، في
`okta-partners/app/Services/Partners/Manifest/MobileEntryRules.php:36`:

```
PATTERN_NATIVE = #^okta_app/native/[A-Za-z0-9_-]+/lib/[A-Za-z0-9._/-]+\.dart$#
```

ويرفض `fits()` أيضاً أي مدخل يحوي `..` (`:83-86`)، ويحدّ الطول بـ 255 (`:39`).

`min_contract` و`capabilities` **خاصّان بـ native**: تبديل الوضع إلى أي شيء آخر
يصفّرهما، في المحرّر وفي أداة MCP على السواء. `[مؤكَّد]`
`okta-partners/app/Services/PartnerMcp/Server/Tools/ControlTools.php:1569-1577`

## القفزة 4 — النشر: okta-web يفحص الـ manifest

`okta-web/app/Modules/Core/ManifestValidator.php`:

| القاعدة | السطر |
|---|---|
| `mobile.mode` ∈ `webview,external,native` | `:446`، `:487` |
| `mobile.minContract` عدد صحيح ≥ 1 | `:453`، `:490` |
| مدخل native يطابق `okta_app/native/<entry>/lib/**.dart` بلا `..` | `:817-821` |
| صلاحيات الجهاز ← `validateDeviceCapabilities()` | `:856`، `:912` |
| الصلاحيات تستلزم وجود سطح native **ما** | `:923-941` |
| الصلاحيات تستلزم `minContract >= DeviceCapability::GATE_CONTRACT` (13) | `:949-958` |
| مدخل تبويب ملف الطالب أيضاً native حصراً، و`minContract >= 17` | `:1000`، `:1024-1041`، `:1120-1125` |

حدّ الصلاحيات موجود لأن البوّابة نفسها لم تكن موجودة قبل العقد 13 — فجهاز تحت
الحدّ كان سيشغّل التطبيق المصغّر على مضيف يتجاهل الموافقة المعلنة بالكامل.
`[مؤكَّد]` `:956-958`

## القفزة 5 — التثبيت: الـ manifest يحطّ على نسخة الجهة

يكتب `PartnerSandboxModuleInstaller` الـ manifest على صفّ `modules` المحلي عند
كل تثبيت، ونقطة الإطلاق تقرأ **ذلك الصفّ** لا مرآة كتالوج الشركاء — فتتبع أحدث
إعداد للشريك دون الاعتماد على مزامنة. `[مؤكَّد]`
`okta-web/app/Http/Controllers/Api/Mobile/MobileAppCatalogController.php:157-165`

## القفزة 6 — الجهاز يطلب إطلاقاً

ينادي `okta-app`، بتوكن Sanctum:

```
POST /api/mobile/app-catalog/{slug}/launch          { tenant_id, role_id }
POST /api/mobile/app-catalog/portal/{slug}/launch   { tenant_id, portal }
```

`okta-web/routes/api.php:119` و`:126`، وكلاهما خلف
`['auth:sanctum', 'mobile.min-version']` (`:117`).

والـ controller:
1. يحمّل الـ `Module` النشط بالـ slug، ويعيد 404 ما لم يكن
   `mobile.supported === true` (`:162-174`)؛
2. يضبط معرّف فريق صلاحيات spatie على الجهة **قبل** ترطيب الدور، فتُقرأ
   الصلاحيات في نطاق الجهة الصحيح (`:178-179`)؛
3. يختار الجمهور المناسب لذلك الدور عبر `NormalizeMobileAudiences::pickForRole`
   (`:185`) — أو `pickForPortal` على مسار البوابة (`:125`)؛
4. يسلّم `IssueWebviewLaunch` كتلةً محلولة تحمل وضع ذلك الجمهور ومدخله
   (`:190-206`).

يتفرّع `IssueWebviewLaunch` على الوضع في
`okta-web/app/Services/MobileAppCatalog/IssueWebviewLaunch.php:51`؛ ويعيد فرع
native (`:173-200`) **رابطاً موقَّعاً قصير العمر** إلى مسار الحمولة، مع
`payload_version` مشتقّاً من طابع `updated_at` للمودول (`:200`).

> **ولهذا لا تحتاج دورة sandbox رفع إصدار.** التثبيت على sandbox يعيد حلّ commit
> الفرع، فيتغيّر `github_commit_sha` على صفّ `modules`، فيتّسخ `$module->update()`،
> فيتحرّك `updated_at` — فيتحرّك `payload_version` ويسقط كاش الجهاز. أمّا إعادة
> التثبيت **بلا دفع** فتترك كل عمود كما هو، فلا شيء يتّسخ، ويبقى الطابع.
> `[مؤكَّد]` `PartnerSandboxModuleInstaller.php:77-87`،
> `okta-partners/app/Services/PartnerModules/Sandbox/InstallVersionOnSandbox.php:140-148`

ويختلف مسارا الإطلاق اختلافاً يستحقّ التسمية: إطلاق البوابة يأخذ
`{tenant_id, portal}` و**لا يأخذ دوراً**، والجهة تأتي من **البطاقة** لا من
الجلسة — فالبوابة تجمع مدارس، والسياق النشط لا جهة فيه ليقرأها. `[مؤكَّد]`
`okta-app/lib/features/miniapps/bridge/okta_dart_miniapp_host.dart:117-122`

## القفزة 7 — الجهاز ينزّل حزمة المصدر

```
GET /app/{slug}/miniapp-payload
```

`okta-web/routes/app.php:45`، خلف
`['web', 'signed', 'app.webview', 'throttle:30,1']` (`:33`).

**التوقيع هو التفويض**؛ وقد أعاد `app.webview` أصلاً ترطيب المستخدم وتأكيد أن
الدور النشط يحمل النطاق المطلوب للبطاقة، فلا يفعل
`MiniappPayloadController` سوى التجميع. `[مؤكَّد]`
`okta-web/app/Http/Controllers/App/MiniappPayloadController.php:12-22`

ويرفض أي شيء وضعه المحلول ليس `native` (`:29-31`)، ثم ينادي
`BundleMiniappSource` (`:36-41`)، ويعيد الـ JSON بترويسة
`Cache-Control: no-store, private` (`:44-45`).

### `BundleMiniappSource` — ما الذي يُقرأ فعلاً

`okta-web/app/Services/MobileAppCatalog/BundleMiniappSource.php`

1. يعيد فحص المدخل مقابل نمط native ويرفض `..` (`:49-55`) — فرضٌ ثانٍ، مستقلّ
   عن بوّابة النشر.
2. يحلّ `Modules/<StudlySlug>/okta_app/native/<entry>/lib` ويطبّق عليه
   `realpath` (`:60-70`).
3. يمشيه تكرارياً آخذاً ملفات `.dart` فقط، **ويسيّج كل ملف**: أي مسار لا يعود
   بـ `realpath` تحت ذلك الـ `lib/` يُتخطّى — وهذا ما يمنع symlink من الهروب
   (`:86-90`).
4. يفرض ثلاثة حدود — 256 كيلوبايت للملف، 2 ميغابايت للحزمة، 128 ملفاً
   (`:32-36`، `:91-97`، `:101-104`).
5. يشترط وجود ملف الدخول المسمّى (`:109-111`).
6. يقرأ اسم حزمة Dart من `pubspec.yaml` الخاص بالحزمة، ويتراجع إلى `mini_app`
   (`:174-188`).
7. يطبّع الصلاحيات المعلنة، مُسقِطاً الأسماء المجهولة والتبريرات الفارغة
   (`:127-168`).

**الحزم الشقيقة لا تُجمَع.** التقاط `<entry>` في الخطوة 1 هو ما يمنع حزمةً من
حمل مصدر أخرى. `[مؤكَّد]` `:26-29`

## القفزة 8 — الجهاز يترجم ويرسم

`okta-app/lib/features/miniapps/bridge/okta_dart_miniapp_host.dart`

- `oktaMiniAppCacheProvider` (`:64-71`) يفتح كاش bytecode دائماً تحت مجلد دعم
  التطبيق.
- `oktaDartBundleProvider` (`:74-95`) **مفتاحه عائلي حسب الهدف**، فبطاقة بوابة
  وبطاقة دور لنفس الـ slug لا تتشاركان جلباً واحداً أبداً.
- `OktaDartMiniApp.build` (`:250-296`) يحلّ الكاش ثم الحزمة، ثم ينادي
  `OktaMiniApp.fromBundle(...)` مع
  `plugins: [OktaHostPlugin(ref.read(oktaHostDelegateProvider(target)))]`
  (`:263-265`).

وداخل `okta-miniapp`:

- `OktaMiniAppBundle.requireSupportedBy` يرمي `OktaMiniAppContractException` حين
  `minContract > hostContract` — **قبل أن يُترجَم أي شيء**
  (`okta-miniapp/lib/src/bundle/okta_mini_app_bundle.dart:154-163`).
- يضيف المترجِم `flutterEvalPlugin` مع `OktaHostPlugin`، الذي يحقن المكتبات
  الأربع في `oktaMiniAppInjectedSources`
  (`okta-miniapp/lib/src/injected_sources.dart:41-46`).
- `OktaHostPlugin.configureForRuntime` يسجّل كل دوال الجسر
  (`okta-miniapp/lib/src/host/okta_host_plugin.dart:79-144`)، ويسلسل بعدها
  `OktaMotionPlugin` و`OktaGlassPlugin` (`:142-143`).
- يعمل البرنامج؛ و`runtime.executeLib(lib, 'main')` يعيد `$Value` قيمته
  `.$value` هي الـ `Widget` الجذر.

ويُخزَّن الـ bytecode تحت `(slug, entry, payloadVersion, runtimeSignature)` —
انظر `data.md` §مفتاح الكاش.

---

## التفرّعات الجانبية (مذكورة، غير متتبَّعة)

| التفرّع | الموضع | ملاحظة |
|---|---|---|
| `mode: webview` | `IssueWebviewLaunch.php:51` | ملفات تحت `okta_app/webview/` تُرسَم في WebView معزول. لا يمكنه إعلان صلاحيات جهاز. |
| `mode: external` | نفسه | صفحة https يستضيفها الشريك. نفس الاستثناء من الصلاحيات. |
| إطلاق البوابة | `MobileAppCatalogController::launchPortal:104` | طالب/وليّ أمر؛ بلا دور، والجهة من البطاقة. تُتبِّع بقدر ما يسمّي الفرق فقط. |
| تبويب ملف الطالب | `ManifestValidator.php:995-1055` | العقد 17. تطبيق مصغّر مركَّب على طالب؛ `Okta.studentHashid()` غير فارغة هنا وحدها. |
| `OktaMotion` / `OktaGlass` | `okta-miniapp/lib/src/motion/`، `lib/src/glass/` | حركة ومادّة يرسمهما المضيف، لأن الصندوق الرملي لا يملك أياً منهما. مجسَّرتان كـ `okta_host`. |
| قرّاء مفاتيح الأدوات | `okta-app/lib/features/toolkeys/` | العقد 12. شكل السحب بالاستطلاع؛ انظر `edges.md`. |
| إخلاء الكاش عند فشل الرسم | `okta_dart_miniapp_host.dart:281-295` | مغطّى في `edges.md`. |
