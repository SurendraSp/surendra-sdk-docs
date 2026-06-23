# Smart Ads Mediation SDK — Changelog

## 1.0.0-RC1 *(2026-06-23)*

Initial release of the AdMob mediation variant of SmartAdsSDK.

### What's included

- **AdMob-backed banner** — adaptive full-width banner using `AdSize.getCurrentOrientationAnchoredAdaptiveBannerAdSize`. Animates in from 0 dp on first fill; impression and click events forwarded to Firebase Analytics.
- **AdMob-backed interstitial** — preloaded interstitial shown when the cadence threshold is met (same threshold logic as the original SDK). Reloads automatically after each display.
- **AdMob mediation waterfall** — FAN and InMobi are supported via optional adapter dependencies (`com.google.ads.mediation:facebook:6.21.0.3`, `com.google.ads.mediation:inmobi:11.3.0.0`). Missing adapters are silently skipped — add only the networks you've configured in the AdMob dashboard.
- **House ads preserved** — Firebase Remote Config house-ad cross-sell logic carries over unchanged. When house ads are available and eligible, they take priority over network interstitials.
- **`ads-sdk-kit` as a transitive dependency** — shared analytics, cadence counters, theming, and house-ad data models are in `io.surendrasp:ads-sdk-kit:1.0.0-RC1`. You do not need to declare it separately.

### Module structure

| Maven artifact | Description |
|---------------|-------------|
| `io.surendrasp:ads-sdk-mediation:1.0.0-RC1` | AdMob mediation layer — declare this in your app |
| `io.surendrasp:ads-sdk-kit:1.0.0-RC1` | Shared utilities — pulled in transitively |

### Migration from `ads-sdk` (FAN-direct)

If you were previously using the original `com.github.SurendraSp:smart-ads` SDK:

1. Replace the dependency with `io.surendrasp:ads-sdk-mediation:1.0.0-RC1`.
2. Update the Maven repository URL to `https://maven.pkg.github.com/SurendraSp/SmartAdsSDK` (same URL, but the group ID changes from `com.github.SurendraSp` to `io.surendrasp`).
3. Replace `AdConfig` with `MediationConfig` — swap FAN placement IDs for AdMob ad unit IDs.
4. Remove any direct FAN SDK dependency from your `build.gradle.kts` — it is no longer needed.
5. Add `<meta-data android:name="com.google.android.gms.ads.APPLICATION_ID" ...>` to `AndroidManifest.xml`.
6. Import from `io.surendrasp.ads.mediation` instead of `io.surendrasp.ads`.

All composables (`AdBannerSlot`, `HouseAdFullScreenHost`, `HouseAdCard`) and the `AdsSDK` entry point keep the same names and signatures — no changes needed in your screen code.
