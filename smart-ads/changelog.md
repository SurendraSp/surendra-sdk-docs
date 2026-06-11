# Smart Ads SDK — Changelog

## 1.0.3

### Breaking changes

- **`AdConfig` simplified** — removed all cadence and timing parameters (`bannerHouseCadence`, `interstitialHouseCadence`, `contentAdInterval`, `interstitialTriggerThreshold`, `interstitialCloseDurationSec`). These are now controlled exclusively via Firebase Remote Config (`ads_cadence_config`). SDK built-in defaults apply when RC has no value. Update your `AdConfig` init to remove these fields.

### New features

- **Round-robin house ads** — `nextAd()` (used by banner and interstitial slots) now cycles through all eligible uninstalled apps in rotation, so each impression shows a different app. Previously it always showed the highest-priority app.
- **Hindi localization** — `HouseAdConfig` has new optional fields `titleHi`, `subtitleHi`, `ctaTextHi`. The SDK picks the right language automatically based on app locale (via `AppCompatDelegate`) then device locale, falling back to English.

### Migration from 1.0.2

Remove cadence params from `AdConfig`:

```kotlin
// Before (1.0.2)
AdConfig(
    fanBannerPlacementId          = "...",
    fanInterstitialPlacementId    = "...",
    bannerHouseCadence            = 50,
    interstitialHouseCadence      = 10,
    contentAdInterval             = 4,
    interstitialTriggerThreshold  = 3,
    interstitialCloseDurationSec  = 5,
    analytics    = ...,
    remoteConfig = ...,
    appPackageId = ...,
)

// After (1.0.3)
AdConfig(
    fanBannerPlacementId       = "...",
    fanInterstitialPlacementId = "...",
    analytics    = ...,
    remoteConfig = ...,
    appPackageId = ...,
)
```

To customize cadence, set `ads_cadence_config` in Firebase Remote Config. See [Remote Config](remote-config.md).

---

## 1.0.2

Initial public release.

### Features

- **`AdsSDK`** — singleton entry point; idempotent `init()`, `onActionCompleted()`, `buildAdList()`, `nextHouseAd()`
- **Banner slot** — `AdBannerSlot` composable with FAN → house-ad fallback; slot API for custom banner UI
- **Interstitial** — `HouseAdFullScreenHost` + `HouseAdFullScreen`; countdown ring, auto-dismiss, app-drop detection; slot API for custom full-screen UI
- **In-feed content ads** — `buildAdList()` injects `HouseAdCard` placeholders at configurable interval
- **House-ad engine** — priority-ordered, installed-app-aware; skips already-installed apps automatically
- **Firebase Remote Config** — live cadence overrides via `house_ads` and `ads_cadence_config` keys; 1-hour fetch interval
- **`AdTheme`** — 6-field theme (`primary`, `onPrimary`, `surface`, `onSurface`, `cardShape`, `ctaShape`); `null` properties fall back to host app's `MaterialTheme`
- **Analytics** — 8 events fired into host app's `FirebaseAnalytics` instance; no separate Firebase project required
- **Facebook Audience Network** — initialized off the main thread to avoid startup latency
- **Compose Stability** — `AdTheme`, `HouseAdConfig`, and `AdConfig` registered in stability config for optimal recomposition skipping

### Requirements

- `minSdk` 26, `compileSdk` 37
- Kotlin 2.x, JDK 21
- Jetpack Compose BOM 2025.x
- Facebook Audience Network 6.21.0
- Firebase BOM 34.14.1
- Coil Compose 2.x (peer dependency, host app provides)
