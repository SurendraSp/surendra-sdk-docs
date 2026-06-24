# Smart Ads Mediation SDK — Changelog

## 1.0.0-RC3 *(2026-06-24)*

### Bug fixes

- **Banner flash with house ads disabled** — when `bannerHouseCadence = -1`, a house ad was briefly visible while the AdMob banner was loading. The house ad composable was incorrectly treated as a loading placeholder. Fixed: house ad now only renders when `AdOrchestrator` explicitly routes to it (`showNetworkBanner = false`). While AdMob is loading the banner area collapses to 0 dp.
- **App Open cooldown logic** — replaced the "minimum background duration" gate with a post-show cooldown. Before the first show there is no restriction — the ad fires whenever the app foregrounds and an ad is ready. After each show a 30-second cooldown is enforced; foreground resumes within that window are skipped silently.

---

## 1.0.0-RC2 *(2026-06-24)*

### New feature — App Open ads

- **App Open ad support** — new optional field `admobAppOpenAdUnitId: String?` in `MediationConfig`. When set, the SDK shows an App Open ad on cold start and every time the app comes to the foreground from background.
- The ad is preloaded on `AdsSDK.init()` and reloaded automatically after each display or after 4-hour expiry (GMA requirement).
- Suppressed automatically if an interstitial is already on screen — no double stacking.
- Impression and click events logged to Firebase Analytics with `slot = "app_open"`, `source = "admob"`.
- Fully opt-in: existing RC1 integrations are unaffected (default is `null` = disabled).

To enable, create an **App open** ad unit in the AdMob console and add:

```kotlin
MediationConfig(
    // ... existing fields ...
    admobAppOpenAdUnitId = "ca-app-pub-XXXXXXXXXXXXXXXX/XXXXXXXXXX",
)
```

---

## 1.0.0-RC1 *(2026-06-23)*

Initial release of the AdMob mediation variant of SmartAdsSDK.

### What's included

- **AdMob-backed banner** — adaptive full-width banner using `AdSize.getCurrentOrientationAnchoredAdaptiveBannerAdSize`. Animates in from 0 dp on first fill; impression and click events forwarded to Firebase Analytics.
- **AdMob-backed interstitial** — preloaded interstitial shown when the cadence threshold is met (same threshold logic as the original SDK). Reloads automatically after each display.
- **AdMob mediation waterfall** — FAN and InMobi are supported via optional adapter dependencies (`com.google.ads.mediation:facebook:6.21.0.3`, `com.google.ads.mediation:inmobi:11.3.0.0`). Missing adapters are silently skipped — add only the networks you've configured in the AdMob dashboard.
- **House ads preserved** — Firebase Remote Config house-ad cross-sell logic carries over unchanged. When house ads are available and eligible, they take priority over network interstitials.
- **`ads-sdk-kit` as a transitive dependency** — shared analytics, cadence counters, theming, and house-ad data models are in `io.surendrasp:ads-sdk-kit:1.0.0-RC2`. You do not need to declare it separately.

### Module structure

| Maven artifact | Description |
|---------------|-------------|
| `io.surendrasp:ads-sdk-mediation:1.0.0-RC2` | AdMob mediation layer — declare this in your app |
| `io.surendrasp:ads-sdk-kit:1.0.0-RC2` | Shared utilities — pulled in transitively |

### Migration from `ads-sdk` (FAN-direct)

If you were previously using the original `com.github.SurendraSp:smart-ads` SDK:

1. Replace the dependency with `io.surendrasp:ads-sdk-mediation:1.0.0-RC2`.
2. Update the Maven repository URL to `https://maven.pkg.github.com/SurendraSp/SmartAdsSDK` (same URL, but the group ID changes from `com.github.SurendraSp` to `io.surendrasp`).
3. Replace `AdConfig` with `MediationConfig` — swap FAN placement IDs for AdMob ad unit IDs.
4. Remove any direct FAN SDK dependency from your `build.gradle.kts` — it is no longer needed.
5. Add `<meta-data android:name="com.google.android.gms.ads.APPLICATION_ID" ...>` to `AndroidManifest.xml`.
6. Import from `io.surendrasp.ads.mediation` instead of `io.surendrasp.ads`.

All composables (`AdBannerSlot`, `HouseAdFullScreenHost`, `HouseAdCard`) and the `AdsSDK` entry point keep the same names and signatures — no changes needed in your screen code.
