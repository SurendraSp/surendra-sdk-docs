# Smart Ads SDK — Changelog

## 1.0.6-RC2

### Bug fixes

- **House ads shown for already-installed apps (Android 11+)** — `HouseAdEngine.isInstalled()` used `PackageManager.getPackageInfo()` which silently throws `NameNotFoundException` on API 30+ for any package not declared in `<queries>`, even if installed. The SDK manifest now declares a `<queries>` intent filter for `ACTION_MAIN + CATEGORY_LAUNCHER`, making all launcher-visible apps queryable. No host-app manifest change required (AAR manifests merge automatically). No Play Store policy concern — `<queries>` intent filters are Google's recommended alternative to the restricted `QUERY_ALL_PACKAGES` permission.
- **Release build failure** — `@Preview` composables in `HouseAdBanner.kt` referenced `androidx.compose.ui.tooling.preview.Preview`, which is a `debugImplementation`-only artifact. Release compilation failed with `Unresolved reference 'tooling'`. Previews are now in a separate `src/debug/` source set file (`HouseAdBannerPreviews.kt`) and are excluded from release builds automatically.

---

## 1.0.6-RC

### New features

- **Progressive ad cadence (ramp for new users)** — the interstitial trigger threshold and content-ad interval now ramp linearly from a gentle starting rate down to the configured target as the user accumulates sessions and actions. Brand-new users see fewer ads; loyal users converge on the steady-state cadence. Configured entirely via Remote Config — see new `*Start`, `loyalSessionCount`, `loyalActionCount`, `sessionDebounceMinutes` keys in [Remote Config](remote-config.md).
- **`AdsSDK.houseAds: StateFlow<List<HouseAdConfig>>`** — live house-ad inventory. Composables that eagerly pick an ad (banner slot, custom mixed lists) should key their `remember` off this so they re-pick when the async RC fetch lands, instead of caching `null` from before the fetch.
- **`AdsSDK.setForceLoyalTier(Boolean)`** — QA / debug override. When `true`, the SDK skips the ramp and applies target cadence regardless of session/action counts. In-memory only; resets on process restart.
- **Auto-debug logging** — when the host app is built with `debuggable=true` (`FLAG_DEBUGGABLE`), the SDK dumps every resolved RC value and the parsed house-ad inventory to logcat under tag `HouseAdRepo`. Release builds stay silent.

### Behaviour changes

- **House ads are now strictly opt-in.** Without an `ads_cadence_config` Remote Config entry, all house-ad cadences default to `-1` (disabled) — every slot routes to FAN automatically. Without a `house_ads` entry, the inventory is empty and the SDK ships with **no** hardcoded fallback ad. Existing apps that set both RC keys behave exactly as before.
- **House-ad cadence defaults removed.** `bannerHouseCadence`, `interstitialHouseCadence`, and `contentAdInterval` no longer default to `50`/`10`/`4` when RC is missing — they default to `-1`. `interstitialTriggerThreshold` still defaults to `3` since it controls *when* an interstitial fires, not whether it's house or FAN.

### Backwards compatibility

- All new RC keys are optional. Omit the `*Start` keys → start defaults to target → no ramp → identical to 1.0.4 behaviour. If you want the ramp, set the `*Start` keys explicitly.
- No source-code changes required for existing host apps — the `AdConfig` API is unchanged.

### UI polish

- `HouseAdBanner` — content row is now properly vertically centered; the "Ad" badge is pinned to the top-left corner instead of stealing horizontal space from the icon. Added `@Preview` composables so the banner renders in Android Studio's preview pane.

---

## 1.0.4

### Bug fixes

- **Banner cadence `0` now correctly stays on house ads** — when `bannerHouseCadence` is `0` (always show house), a FAN banner load no longer counts as an impression or resets the slot back to FAN. The slot stays on the house ad as long as an eligible app exists. If all apps are installed, it falls back to FAN with a `house_ad_skipped` event.
- **House banner shown with cadence `0` no longer resets to FAN** — `onHouseBannerShown` now keeps `showFanBanner = false` when cadence is `0`, so consecutive impressions stay on house ads.
- **`contentAdInterval = 0` is now treated as disabled** — previously `0` was a valid "always inject" value; it now behaves the same as `-1` (no content ads). Only `N > 0` enables injection.

---

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
