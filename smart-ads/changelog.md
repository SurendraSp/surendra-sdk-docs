# Smart Ads SDK — Changelog

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
