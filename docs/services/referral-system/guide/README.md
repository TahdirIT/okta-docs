# دليل نظام الإحالات — محرّك المكافآت ودورة الحياة

## دورة حياة الإحالة

```
pending ─▶ qualified ─▶ rewarded
   │           │
   │           └─(rejected بعد ~90 يوماً إن بقيت pending)
   └─ flagged (risk_score ≥ 70)
```

1. **الإنشاء**: عند التسجيل، `UserObserver::created` ينشئ صفّ `Referral`
   (`pending`)، يربط `parent_referral_id` بإحالة المُحيل المؤهَّلة/المكافأة (سلسلة
   المستويات)، يشغّل `EvaluateReferralRisk`، ويطلق `ReferralCreated`.
2. **التأهيل**: حسب `referral_settings.qualification_trigger` (افتراضي
   `first_subscription`): `SubscriptionPaid`، أو `time_based` عبر المجدول
   (`time_based_days_after_registration` افتراضي 30 يوماً)، أو اعتماد إداري في
   `ApprovalsPage` — كلها تضبط `qualified` + `qualified_at` وتطلق `ReferralQualified`.
3. **الصرف**: `ReferralQualified → DistributeMultiLevelRewards` — في معاملة DB،
   يمشي `parent` حتى `max_depth`، يحسب مبلغ كل مستوى، ويستدعي `ApplyRewardToWallet`.
4. `ReferralRewarded → CheckBadgesAndLevelsOnRewarded` (تلعيب).

## محرّك المكافآت متعدّد المستويات

مُعرَّف بمخزن `referral_settings` (لا بأعمدة): `CalculateReferrerReward` يقرأ
`referrer.reward_type` (`fixed`/`percentage`)، `reward_amount`/`reward_percentage`،
و`multilevel.percentages` (مثال `[100, 20, 5]` = نِسَب L1/L2/L3). مُضاعِف المستوى N
= `percentages[N-1]/100`؛ إن كان `multilevel.enabled=false` يدفع L1 فقط.
`multilevel.max_depth` (افتراضي 3) يحدّ المشي، و`referral.level` يُضبط عند الإنشاء
بـ `min(3, parent.level+1)`.

`ApplyRewardToWallet` **idempotent**: يقفل المكافأة القائمة لكل
(`referral`+`beneficiary`+`type`)، يتخطّى إن كانت `credited`، ينشئ `ReferralReward`
`pending` بلقطة إعدادات كاملة، ثم يضيف للمحفظة عبر `CreditWallet` (مصدر
`referral_reward`) ويقلبها `credited`. مكافآت `discount` تبقى `pending` بلا إضافة محفظة.

**مسار المؤثّرين منفصل**: `UserObserver` يسجّل `InfluencerConversion` وقد يمنح
المُحال بونص `fixed_signup` فوراً (مصدر `signup_bonus`).

## مكافحة الاحتيال

`EvaluateReferralRisk` يجمع إشارات ← يحسب درجة ← يوسم:

| الإشارة | الوزن |
|---|---|
| self_referral_identity (المُحيل=المُحال) | 100 |
| shared_fingerprint_with_referrer | 50 |
| shared_ip_with_referrer | 40 |
| ip_rate_limit_exceeded | 35 |
| daily_limit_exceeded | 30 |
| disposable_email | 25 |
| matching_email_domain | 15 |
| missing_otp_verification | 10 |

الدرجة محصورة 0–100، وعتبة الوسم التلقائي **≥70** (فإشارة ضعيفة واحدة لا توسِم).
`FlagSuspiciousReferral` يكتب `risk_score`/`flagged_reason` ويضبط `flagged`.
حدود المعدّل (`CheckIpRateLimit`/`CheckDailyLimit`) إشارات لا حجب صارم. OTP عبر
`RequestReferralOtp`/`VerifyReferralOtp` يختم `otp_verified_at` فتسقط إشارة نقص OTP.
