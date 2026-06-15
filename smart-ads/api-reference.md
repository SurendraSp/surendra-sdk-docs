# Smart Ads SDK — API Reference

All public symbols in `io.surendrasp.ads`. Internal classes (`AdOrchestrator`, `AdCounters`, `HouseAdEngine`, `HouseAdRepository`, `FanBannerManager`, `FanInterstitialManager`, `UserLifecycle`, `CadenceResolver`) are not part of the public API.

---

## `AdsSDK` (object)

The singleton entry point. All interaction happens through this object.

```kotlin
object AdsSDK
```

### Methods

#### `init(context, config)`

```kotlin
fun init(context: Context, config: AdConfig)
```

Initializes the SDK. Must be called once in `Application.onCreate()` before any other SDK call. Subsequent calls are no-ops (idempotent).

Internally:
- Initializes Facebook Audience Network on a background IO thread (avoids ~50 ms startup cost on main thread).
- Builds the internal `AdOrchestrator`, `HouseAdEngine`, `HouseAdRepository`, `FanBannerManager`, `FanInterstitialManager`, and `UserLifecycle`.
- Starts a Firebase Remote Config fetch (1-hour interval).
- Counts this init as a session start (debounced — see `sessionDebounceMinutes` in [Remote Config](remote-config.md)).
- Detects host-app debug flag (`ApplicationInfo.FLAG_DEBUGGABLE`); when set, the SDK dumps every resolved RC value to logcat after fetch under tag `HouseAdRepo`.

| Parameter | Type | Description |
|-----------|------|-------------|
| `context` | `Context` | Use `applicationContext` |
| `config` | `AdConfig` | Full SDK configuration |

---

#### `onActionCompleted(context)`

```kotlin
fun onActionCompleted(context: Context)
```

Signal that the user completed a qualifying action (save, download, share, export, etc.). The SDK increments two counters: an internal threshold counter, and a persisted lifetime action counter used by the progressive cadence ramp. An interstitial fires after every `interstitialTriggerThreshold` calls — the effective threshold ramps from `interstitialTriggerThresholdStart` down to `interstitialTriggerThreshold` as the user accumulates sessions/actions. See [Remote Config](remote-config.md#how-the-ramp-works) for the math.

**Decision tree when threshold is reached:**
1. If it's the house-ad interstitial turn AND a house ad is eligible → show house-ad full screen, preload FAN for next time.
2. If FAN interstitial is ready → show FAN interstitial.
3. FAN not ready AND house ads not disabled → show house-ad full screen as fallback.
4. Nothing available → silently skip.

Counter resets to zero each time the threshold is hit.

---

#### `buildAdList(items, context)`

```kotlin
fun <T> buildAdList(items: List<T>, context: Context): List<AdListItem<T>>
```

Builds a mixed list with house-ad placeholders injected at the *effective* `contentAdInterval`. The interval ramps from `contentAdIntervalStart` to `contentAdInterval` based on user engagement (see [Remote Config](remote-config.md#how-the-ramp-works)). Default is `-1` (disabled) when no RC is configured.

| Effective `contentAdInterval` | Behaviour |
|-------------------------------|-----------|
| `-1` or `0` | Returns `items` unchanged — no ads injected |
| `N > 0` | 1 house ad after every N real items |

> **Important — re-build when inventory arrives.** RC fetch is async. On the first composition the inventory is empty and this method returns `items` unchanged. Consumers that call `buildAdList` inside `remember(items)` must include `AdsSDK.houseAds` as a key, otherwise the empty result is cached forever. See [houseAds](#houseads).

The eligible house-ad queue is built from currently uninstalled apps, in priority order. If the queue is exhausted before all injection points are filled, remaining slots are silently skipped (no gap in the list).

Returns `List<AdListItem<T>>` — see [AdListItem](#adlistitem) below.

---

#### `nextHouseAd(context)`

```kotlin
fun nextHouseAd(context: Context): HouseAdConfig?
```

Returns the next house ad in round-robin order among uninstalled apps, or `null` if all apps are installed. Used internally by `AdBannerSlot`; call directly only if building fully custom UI.

---

### Properties

#### `showFanBanner`

```kotlin
val showFanBanner: StateFlow<Boolean>
```

`true` when the FAN banner should be visible. Collected by `AdBannerSlot`. Becomes `false` when FAN fails and no house ad is available (slot hides entirely). Resets to `true` after a house-ad banner impression.

---

#### `showHouseAdFullScreen`

```kotlin
val showHouseAdFullScreen: SharedFlow<HouseAdConfig>
```

Emits a `HouseAdConfig` each time the orchestrator decides a house-ad interstitial should be shown. Collected by `HouseAdFullScreenHost`. `extraBufferCapacity = 1` — one pending emission is allowed.

---

#### `interstitialCloseDurationSec`

```kotlin
val interstitialCloseDurationSec: StateFlow<Int>
```

Seconds before the close button appears on the house-ad full-screen. Starts at the SDK default (5); updated live when Firebase Remote Config fetch completes (`ads_cadence_config.interstitialCloseSec`).

---

#### `theme`

```kotlin
val theme: AdTheme
```

The `AdTheme` passed in `AdConfig`. Available after `init()`. Used by the SDK's default composables; also available to host app code that needs to match SDK colors.

---

#### `houseAds`

```kotlin
val houseAds: StateFlow<List<HouseAdConfig>>
```

Live house-ad inventory parsed from `house_ads` Remote Config. Starts as an empty list and updates when the async RC fetch completes.

Composables that eagerly pick an ad (banner slot, mixed content list) **must** use this as a `remember` key — otherwise they cache `null` from before the fetch landed and never recover:

```kotlin
val houseAds by AdsSDK.houseAds.collectAsStateWithLifecycle()
val mixedList = remember(items, houseAds) { AdsSDK.buildAdList(items, context) }
```

`AdBannerSlot` does this automatically.

---

### `setForceLoyalTier(enabled)`

```kotlin
fun setForceLoyalTier(enabled: Boolean)
```

QA / debug override. When `true`, the progressive-cadence resolver treats the user as fully loyal (`progress = 1.0`), so the effective interstitial threshold and content-ad interval immediately collapse to their RC target values. Useful for verifying target-rate UX without waiting for a real user to graduate.

In-memory only — resets to `false` on process restart. Safe to call before `init()`; the flag will apply once the SDK is initialized.

---

## `AdConfig` (data class)

```kotlin
data class AdConfig(
    val fanBannerPlacementId:        String,
    val fanInterstitialPlacementId:  String,
    val analytics:                   FirebaseAnalytics,
    val remoteConfig:                FirebaseRemoteConfig,
    val appPackageId:                String,
    val theme:                       AdTheme = AdTheme(),
)
```

All cadence and timing values are controlled via Firebase Remote Config key `ads_cadence_config`. See [Remote Config](remote-config.md) for the full schema and SDK built-in defaults.

| Parameter | Type | Description |
|-----------|------|-------------|
| `fanBannerPlacementId` | `String` | FAN banner placement ID |
| `fanInterstitialPlacementId` | `String` | FAN interstitial placement ID |
| `analytics` | `FirebaseAnalytics` | Host app's Firebase Analytics instance |
| `remoteConfig` | `FirebaseRemoteConfig` | Host app's Firebase Remote Config instance |
| `appPackageId` | `String` | Host app package name (tagged on all analytics events) |
| `theme` | `AdTheme` | Optional brand theming; see [Theming](theming.md) |

---

## `AdTheme` (data class)

```kotlin
data class AdTheme(
    val primary:   Color? = null,
    val onPrimary: Color? = null,
    val surface:   Color? = null,
    val onSurface: Color? = null,
    val cardShape: Shape? = null,
    val ctaShape:  Shape? = null,
)
```

| Field | Applied to | Fallback |
|-------|-----------|---------|
| `primary` | CTA button background, icon placeholder tint | `MaterialTheme.colorScheme.primary` |
| `onPrimary` | CTA button text / icon | `MaterialTheme.colorScheme.onPrimary` |
| `surface` | Banner background, card gradient | `MaterialTheme.colorScheme.secondaryContainer` |
| `onSurface` | Title, subtitle, ad-badge text | `MaterialTheme.colorScheme.onSecondaryContainer` |
| `cardShape` | Content card corner radius | `RoundedCornerShape(12.dp)` |
| `ctaShape` | CTA button corner radius | `RoundedCornerShape(8.dp)` |

Companion constants:
```kotlin
AdTheme.DEFAULT_CARD_SHAPE  // RoundedCornerShape(12.dp)
AdTheme.DEFAULT_CTA_SHAPE   // RoundedCornerShape(8.dp)
```

See [Theming](theming.md) for full details and examples.

---

## `HouseAdConfig` (data class)

```kotlin
data class HouseAdConfig(
    val packageName:  String,
    val title:        String,
    val titleHi:      String = "",
    val subtitle:     String = "",
    val subtitleHi:   String = "",
    val iconUrl:      String = "",
    val playStoreUrl: String,
    val ctaText:      String = "Install",
    val ctaTextHi:    String = "",
    val priority:     Int    = 0,
)
```

Deserialized from the `house_ads` Firebase Remote Config key. Passed to slot composables and `AdListItem.HouseAd`.

| Field | Description |
|-------|-------------|
| `packageName` | Android package name — used for installed-app check |
| `title` | App name (English) |
| `titleHi` | App name (Hindi). Empty → falls back to `title` |
| `subtitle` | Short tagline (English) |
| `subtitleHi` | Short tagline (Hindi). Empty → falls back to `subtitle` |
| `iconUrl` | App icon URL; empty string → placeholder shown |
| `playStoreUrl` | Full Play Store URL opened by the CTA button |
| `ctaText` | CTA button label (English, default `"Install"`) |
| `ctaTextHi` | CTA button label (Hindi). Empty → falls back to `ctaText` |
| `priority` | Lower = shown first; assigned from JSON array index during parse |

### Locale resolution methods

```kotlin
fun resolvedTitle(context: Context): String
fun resolvedSubtitle(context: Context): String
fun resolvedCtaText(context: Context): String
```

Call these instead of accessing `title`, `subtitle`, `ctaText` directly when rendering ad UI. Each method returns the Hindi string when the device/app locale is `hi`, or falls back to the English field. Resolution order:

1. App-level locale set via `AppCompatDelegate.setApplicationLocales`
2. Device system language (`Locale.getDefault().language`)
3. English fallback (the base field)

---

## `AdListItem<T>` (sealed interface)

```kotlin
sealed interface AdListItem<out T> {
    data class Content<T>(val item: T)            : AdListItem<T>
    data class HouseAd(val config: HouseAdConfig) : AdListItem<Nothing>
}
```

Returned by `AdsSDK.buildAdList()`. Use in a `LazyColumn` / `LazyVerticalGrid`:

```kotlin
when (item) {
    is AdListItem.Content -> YourContentCard(item.item)
    is AdListItem.HouseAd -> HouseAdCard(item.config)   // or your own card
}
```

---

## Composables

### `AdBannerSlot`

```kotlin
@Composable
fun AdBannerSlot(
    modifier:       Modifier = Modifier,
    houseAdContent: (@Composable (config: HouseAdConfig) -> Unit)? = null,
)
```

Drop-in banner slot. Collects `AdsSDK.showFanBanner` and renders either:
- FAN banner (`AdView` via `AndroidView`) when `showFanBanner == true`
- House-ad banner when FAN fails or cadence dictates

The FAN `AdView` is created once and visibility-toggled (not recomposed) to prevent flicker.

Sets `LocalAdTheme` for all nested SDK composables.

**Slot API:** pass `houseAdContent` to replace the SDK default `HouseAdBanner` with your own composable. The SDK calls it with the current `HouseAdConfig`.

```kotlin
AdBannerSlot(
    houseAdContent = { config ->
        MyCustomBanner(config)   // any height / style
    }
)
```

---

### `HouseAdFullScreenHost`

```kotlin
@Composable
fun HouseAdFullScreenHost(
    content: (@Composable (config: HouseAdConfig, onDismiss: () -> Unit) -> Unit)? = null,
)
```

Transparent host composable. Collects `AdsSDK.showHouseAdFullScreen` and renders the interstitial overlay when it emits.

**Placement:** must be at the root of your screen composable, **outside** `Scaffold`, so it overlays all content.

**Slot API:** pass `content` to fully replace the SDK default `HouseAdFullScreen`. The SDK handles timing and analytics; you own the pixels. **You must call `onDismiss` when the user closes the ad.**

```kotlin
HouseAdFullScreenHost(
    content = { config, onDismiss ->
        MyFullScreenAd(config = config, onClose = onDismiss)
    }
)
```

---

### `HouseAdBanner`

```kotlin
@Composable
fun HouseAdBanner(
    config:   HouseAdConfig,
    modifier: Modifier = Modifier,
    onShown:  () -> Unit = {},
)
```

SDK default banner UI. 50 dp tall (matches FAN banner height). Layout: vertically-centered `[icon] [title/subtitle] [CTA button]` row, with a small "Ad" badge floating in the top-left corner. CTA opens `config.playStoreUrl` via `Intent.ACTION_VIEW`. Calls `onShown()` via `LaunchedEffect` on first composition.

Themed via `LocalAdTheme`. Includes `@Preview` composables for inspecting layout in Android Studio without running the app.

---

### `HouseAdCard`

```kotlin
@Composable
fun HouseAdCard(
    config:   HouseAdConfig,
    modifier: Modifier = Modifier,
)
```

SDK default in-grid content card. Default aspect ratio 1:1 (square). Gradient background from `surface` → `secondaryContainer`. Entire card is clickable (opens Play Store). Override `modifier` to use a non-square aspect ratio for list layouts.

Themed via `LocalAdTheme`.

---

### `HouseAdFullScreen`

```kotlin
@Composable
fun HouseAdFullScreen(
    config:           HouseAdConfig,
    closeDurationSec: Int,
    onDismiss:        () -> Unit,
)
```

SDK default full-screen interstitial overlay. Dark semi-transparent background (93% black). Behaviour:

- Animated countdown ring replaces close button for `closeDurationSec` seconds.
- After countdown → close button fades in; user must tap to dismiss.
- If countdown completes without user tap → auto-dismissed + fires `interstitial_timed_close` event.
- If app goes to background while visible → fires `interstitial_app_drop`; does not call `onDismiss` (ad stays ready when user returns).
- On `ON_RESUME` after background → calls `onDismiss` to cleanly remove the ad.
- Visible seconds are tracked and reported in all interstitial analytics events.
- Tapping anywhere on the screen (outside the close button area) fires the CTA and opens the Play Store.

---

## Analytics events

All events are fired into the host app's `FirebaseAnalytics` instance (passed in `AdConfig`). The SDK owns no Firebase project — events appear in the host app's console.

| Event name | Parameters | Fired when |
|------------|-----------|-----------|
| `ad_impression` | `slot`, `source`, `app_package_id` | Any ad impression (banner or house) |
| `ad_click` | `slot`, `source`, `app_package_id` | User taps an ad CTA |
| `ad_load_failed` | `slot`, `error_code`, `error_message`, `app_package_id` | FAN banner fails to load |
| `house_ad_skipped` | `slot`, `app_package_id` | House ad was eligible but skipped (no uninstalled app) |
| `interstitial_shown` | `source` (`"fan"` \| `"house"` \| `"house_fallback"`), `app_package_id` | Interstitial becomes visible |
| `interstitial_dismissed` | `source`, `visible_seconds`, `app_package_id` | User taps X to close house-ad interstitial |
| `interstitial_timed_close` | `source`, `visible_seconds`, `app_package_id` | Close button enabled after countdown (may still be on screen) |
| `interstitial_app_drop` | `source`, `visible_seconds`, `app_package_id` | User backgrounded app while interstitial was showing |
