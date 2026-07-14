# الطبقة التقنية — نظام الإحالات

- [نماذج البيانات](./models/README.md) — جداول landlord: الإحالة، المكافآت، المحافظ، المستويات/الأوسمة، المؤثّرون.
- [وظائف الخدمة (Use-cases)](./service-functions/README.md) — الأكواد، المكافآت، المحافظ، المؤثّرون، التلعيب، الأمان، دورة الحياة.

**الأحداث** (تُسجَّل من `ReferralServiceProvider::registerEvents`):
`ReferralCreated → ReferralQualified → ReferralRewarded`، تربط
`UserObserver` بـ `EvaluateReferralRisk` ثم `DistributeMultiLevelRewards` ثم
`CheckBadgesAndLevelsOnRewarded`.

**Middleware**: `referral.capture` (`CaptureReferralCode`) — يقرأ `?ref=CODE`
(regex `^[A-Z0-9]{4,10}$`)، يتحقّق من `referral_codes` النشطة (أو `influencers`)،
يزيد `clicks_count`، ويضع كوكي `referral_code` موقَّعاً httpOnly (TTL افتراضي 30
يوماً) يستهلكه `UserObserver` عند التسجيل.

**الصلاحيات** (نطاق system، عبر `ReferralPermissionsSeeder`، تنسال لـ superadmin):
- مستخدم: `referrals.codes.{view,regenerate}`، `referrals.wallet.{view,withdraw}`، `referrals.history.view`، `referrals.badges.view`.
- إدارة: `referrals.admin.dashboard.view`، `referrals.admin.referrals.{view,approve,reject,clawback,recompute,view_suspicious}`، `referrals.admin.wallets.{view,adjust}`، `referrals.admin.settings.{view,update}`، `referrals.admin.reports.{view,export}`، `referrals.admin.badges.*`، `referrals.admin.levels.update`، `referrals.admin.currencies.{view,update}`، `referrals.admin.influencers.*`.
