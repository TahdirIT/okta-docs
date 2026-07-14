# وظائف الخدمة (Use-cases)

كل use-case `final class` بـ `__invoke()` تحت `app/Services/ReferralSystem/<Group>/`.

## Codes
`GenerateReferralCode` (كود فريد لكل مستخدم عند التسجيل) · `RegenerateReferralCode` ·
`ValidateReferralCode`.

## Rewards
`CalculateReferrerReward` (المبلغ الأساس × مُضاعِف المستوى) · `CalculateReferredReward` ·
`DistributeMultiLevelRewards` (يمشي سلسلة `parent`) · `ApplyRewardToWallet`
(إضافة idempotent) · `RecomputeReferralRewards` (إعادة تشغيل إدارية).

## Wallets
`CreditWallet` · `DebitWallet` · `GetBalance` · `FreezeAmount` · `UnfreezeAmount` ·
`ConvertCurrency` (عبر `currency_exchange_rates`).

## Influencers
`CreateInfluencer` · `UpdateInfluencer` · `DeleteInfluencer` · `ToggleInfluencerActive` ·
`GenerateInfluencerCode` · `RecordInfluencerClick` · `RecordInfluencerConversion` ·
`CreditInfluencerWallet` · `DebitInfluencerWallet` · `GetInfluencerStats`.

## Gamification
`BackfillUserAchievements` (اشتقاق الأوسمة/المستويات من الإحالات المؤهَّلة/المكافأة).

## Security (مكافحة الاحتيال)
`EvaluateReferralRisk` (المنسّق) · `CalculateRiskScore` · `DetectSelfReferral` ·
`CheckIpRateLimit` · `CheckDailyLimit` · `FlagSuspiciousReferral` ·
`RequestReferralOtp` / `VerifyReferralOtp` (عبر `App\Services\Otp\OtpService`).

## Lifecycle (مجدولة)
`QualifyTimeBasedReferrals` (`referrals:qualify-time-based`، `0 2 * * *`) ·
`ExpireStaleReferrals` (`referrals:expire-stale`، `0 3 * * *`) ·
`GenerateDailyReferralReport` (`referrals:daily-report`، `30 3 * * *`).

## Reports · Settings
`GetReferralDashboardSummary` · `GetActiveReferralSettings` / `UpdateReferralSettings`
(مخزن KV على `referral_settings`).
