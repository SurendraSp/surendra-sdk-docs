# Smart Ads Mediation SDK — API Reference

**Package:** `io.surendrasp.ads.mediation`  
**Maven:** `io.surendrasp:ads-sdk-mediation:1.0.0-RC3`

---

## AdsSDK

Singleton entry point. All methods are thread-safe.

```kotlin
object AdsSDK
```

### init

```kotlin
fun init(context: Context, config: MediationConfig)
```

Initializes the SDK. Must be called from `Application.onCreate()`. Subsequent calls are no-ops.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `context` | Application or Activity context — the SDK stores `applicationContext` internally. |
| `config` | `MediationConfig` instance with ad unit IDs and Firebase references. |

Internally this calls `MobileAds.initialize(context)` on the calling thread, then initializes the orchestrator and lifecycle observer.

---

### onActionCompleted

```kotlin
fun onActionCompleted(context: Context)
```

Signal that a qualifying user action occurred (save, export, download, share, etc.). The SDK increments the action counter and, if the cadence threshold is met, shows an ad.

- If a house ad is available (from Remote Config), it shows a house ad full-screen overlay.
- Otherwise it fires a preloaded AdMob interstitial.

Safe to call on any thread — the SDK hops to the main thread internally if needed.

---

### houseAds

```kotlin
val houseAds: StateFlow<List<HouseAdConfig>>
```

Emits the current list of house ads fetched from Firebase Remote Config. Collect this in your composables with `collectAsStateWithLifecycle()`.

---

### buildAdList

```kotlin
fun <T> buildAdList(items: List<T>, context: Context): List<AdListItem<T>>
```

Interleaves house-ad placeholders into `items` at the cadence interval defined in Remote Config. Returns a mixed list of `AdListItem.Content` and `AdListItem.HouseAd` entries for use in a `LazyColumn` or `LazyVerticalGrid`.

---

## MediationConfig

Data class holding all configuration for the mediation SDK.

```kotlin
data class MediationConfig(
    val admobBannerAdUnitId: String,
    val admobInterstitialAdUnitId: String,
    val admobAppOpenAdUnitId: String? = null,
    val analytics: FirebaseAnalytics,
    val remoteConfig: FirebaseRemoteConfig,
    val appPackageId: String,
    val theme: AdTheme = AdTheme(),
)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `admobBannerAdUnitId` | `String` | — | AdMob banner ad unit ID (`ca-app-pub-…/…`) |
| `admobInterstitialAdUnitId` | `String` | — | AdMob interstitial ad unit ID |
| `admobAppOpenAdUnitId` | `String?` | `null` | AdMob App Open ad unit ID; `null` disables App Open ads entirely |
| `analytics` | `FirebaseAnalytics` | — | Firebase Analytics instance from the host app |
| `remoteConfig` | `FirebaseRemoteConfig` | — | Firebase Remote Config instance from the host app |
| `appPackageId` | `String` | — | Host app package name — tagged on all analytics events |
| `theme` | `AdTheme` | `AdTheme()` | Brand colors and shapes for house-ad UI; falls back to MaterialTheme values if not set |

---

## Composables

### AdBannerSlot

```kotlin
@Composable
fun AdBannerSlot(modifier: Modifier = Modifier)
```

Renders an adaptive full-width AdMob banner. The ad size is computed via `AdSize.getCurrentOrientationAnchoredAdaptiveBannerAdSize` using the current screen width, so the banner fills the available width and adjusts height automatically on orientation changes.

The composable animates in from 0 dp height when the first ad loads. Before the ad loads, it occupies zero space (no placeholder gap).

**Impression and click events** are forwarded to Firebase Analytics automatically (`AdEvent.Impression` / `AdEvent.Click` with `slot = "banner"`, `source = "admob"`).

> Add this composable to screens that should show a persistent banner. You do not need to call any SDK method to load or refresh the banner — it manages its own lifecycle.

---

### HouseAdFullScreenHost

```kotlin
@Composable
fun HouseAdFullScreenHost()
```

Displays the house-ad full-screen overlay when triggered. Place this **once per screen**, outside `Scaffold`, so it overlays the entire screen including the navigation bar and status bar areas.

This composable observes the internal overlay state — it renders nothing until `onActionCompleted` triggers a house ad.

---

### HouseAdCard

```kotlin
@Composable
fun HouseAdCard(config: HouseAdConfig, modifier: Modifier = Modifier)
```

Renders a single house-ad card for use in a content feed. Typically used inside the `AdListItem.HouseAd` branch of `buildAdList()` output.

---

## AdListItem

Sealed class returned by `buildAdList()`.

```kotlin
sealed class AdListItem<out T> {
    data class Content<T>(val item: T) : AdListItem<T>()
    data class HouseAd<T>(val config: HouseAdConfig) : AdListItem<T>()
}
```

---

## AdTheme

Controls the visual appearance of house-ad UI. All fields are optional — unset fields fall back to the host app's MaterialTheme.

```kotlin
data class AdTheme(
    val primary: Color? = null,
    val onPrimary: Color? = null,
    val surface: Color? = null,
    val onSurface: Color? = null,
    val cardShape: Shape? = null,
    val ctaShape: Shape? = null,
)
```

See [Theming](../smart-ads/theming.md) for the full MaterialTheme fallback table and usage examples.

---

## HouseAdConfig

Describes a single house ad loaded from Firebase Remote Config.

```kotlin
data class HouseAdConfig(
    val title: String,
    val subtitle: String,
    val ctaText: String,
    val iconUrl: String,
    val targetPackageId: String,
    val targetDeepLink: String? = null,
)
```

| Field | Description |
|-------|-------------|
| `title` | Primary ad headline |
| `subtitle` | Supporting copy |
| `ctaText` | Call-to-action button label |
| `iconUrl` | App icon URL (loaded via Coil) |
| `targetPackageId` | Package name to open on tap; used to detect if the app is already installed |
| `targetDeepLink` | Optional deep link URI; if provided, used in place of a Play Store launch |

---

## Analytics events

All events are logged to Firebase Analytics. The `appPackageId` field on every event comes from `MediationConfig.appPackageId`.

| Event class | Logged when |
|-------------|-------------|
| `AdEvent.Impression` | A banner, interstitial, or App Open ad becomes visible |
| `AdEvent.Click` | The user taps a banner, interstitial, App Open ad, or house-ad CTA |
| `AdEvent.Conversion` | The house-ad target app is opened (CTA tapped + navigation succeeded) |
| `AdEvent.LoadFailed` | An ad fails to load (any format) |
| `AdEvent.InterstitialShown` | An interstitial is shown (network or house) |
| `AdEvent.InterstitialDismissed` | An interstitial is dismissed |

The `source` field is `"admob"` for network ads and `"house"` for house ads. The `slot` field is `"banner"`, `"interstitial"`, `"app_open"`, or `"content"`.
