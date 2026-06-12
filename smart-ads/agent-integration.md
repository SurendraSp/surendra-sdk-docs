# Smart Ads SDK — Agent Integration Guide

This document is a self-contained reference for AI agents integrating the Smart Ads SDK into an Android app. Every public class, method, RC key, and composable is documented here. Follow the checklist at the end to verify a complete integration without asking any follow-up questions.

---

## Identity

| | |
|---|---|
| **Library** | Smart Ads SDK |
| **Maven coordinate** | `com.github.SurendraSp:smart-ads:1.0.4` |
| **GitHub Packages URL** | `https://maven.pkg.github.com/SurendraSp/SmartAdsSDK` |
| **Root package** | `io.surendrasp.ads` |
| **Current version** | `1.0.4` |
| **minSdk** | 26 |
| **compileSdk** | 37 |
| **Kotlin** | 2.x |
| **JDK** | 21 |
| **Compose BOM** | 2025.x |

---

## What the SDK does

The SDK manages three ad surfaces for Android Compose apps:

1. **Banner** — A 50 dp horizontal banner. Shows Facebook Audience Network (FAN) ad; falls back to a cross-sell house ad when FAN fails or cadence dictates.
2. **Interstitial** — A full-screen overlay triggered after qualifying user actions (saves, downloads, shares). FAN interstitial takes priority; house-ad full-screen is the fallback.
3. **In-feed content ads** — House-ad cards injected into content lists/grids at a configurable interval.

"House ads" are cross-sell promotions for other apps in the same portfolio. They are configured via Firebase Remote Config and shown only to users who do not have the advertised app installed.

---

## Step-by-step integration

### Step 1 — Add Maven repository

In `settings.gradle.kts`, inside `dependencyResolutionManagement.repositories`:

```kotlin
maven {
    name = "SmartAdsSDK"
    url  = uri("https://maven.pkg.github.com/SurendraSp/SmartAdsSDK")
    credentials {
        username = providers.gradleProperty("gpr.user").orNull
            ?: System.getenv("GPR_USER")
        password = providers.gradleProperty("gpr.key").orNull
            ?: System.getenv("SMART_ADS_SDK_GPR_KEY")
    }
}
```

Credentials come from one of three sources (in priority order):
1. `local.properties` keys `gpr.user` and `gpr.key`
2. Environment variables `GPR_USER` and `SMART_ADS_SDK_GPR_KEY`
3. `gh auth token` (GitHub CLI — used automatically as a fallback by the SDK's own build, not by consuming apps)

For CI, set `GPR_USER` and `SMART_ADS_SDK_GPR_KEY` as secrets. The PAT needs `read:packages` scope.

### Step 2 — Add dependencies

In the app module's `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.github.SurendraSp:smart-ads:1.0.4")

    // Required peer dependencies
    implementation("io.coil-kt:coil-compose:2.7.0")                            // image loading
    implementation(platform("com.google.firebase:firebase-bom:34.14.1"))
    implementation("com.google.firebase:firebase-analytics")
    implementation("com.google.firebase:firebase-config")

    // If not already present:
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.x.x")       // collectAsStateWithLifecycle
}
```

**Do NOT add `com.facebook.android:audience-network-sdk`** — the SDK bundles it.

### Step 3 — Initialize in Application subclass

Create `MyApp : Application()` if one does not exist:

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        AdsSDK.init(
            context = applicationContext,
            config  = AdConfig(
                fanBannerPlacementId       = "YOUR_FAN_BANNER_PLACEMENT_ID",
                fanInterstitialPlacementId = "YOUR_FAN_INTERSTITIAL_PLACEMENT_ID",
                analytics    = FirebaseAnalytics.getInstance(this),
                remoteConfig = Firebase.remoteConfig,
                appPackageId = BuildConfig.APPLICATION_ID,
                theme = AdTheme(
                    primary   = Color(0xFF1B8C3E),
                    onPrimary = Color.White,
                ),
            )
        )
    }
}
```

> Cadence and timing values (`bannerHouseCadence`, `interstitialHouseCadence`, `contentAdInterval`, `interstitialTriggerThreshold`, `interstitialCloseDurationSec`) are controlled exclusively via the `ads_cadence_config` Firebase Remote Config key. SDK built-in defaults apply when RC has no value. See the RC keys section below.
```

Register in `AndroidManifest.xml`:

```xml
<application android:name=".MyApp" ...>
```

`AdsSDK.init()` is idempotent — multiple calls are safe.

### Step 4 — Add banner

Place `AdBannerSlot()` in any screen that needs a banner. Typically at the bottom:

```kotlin
import io.surendrasp.ads.ui.AdBannerSlot

@Composable
fun SomeScreen() {
    Column(modifier = Modifier.fillMaxSize()) {
        // ... screen content ...
        AdBannerSlot(modifier = Modifier.fillMaxWidth())
    }
}
```

### Step 5 — Add interstitial host

Place `HouseAdFullScreenHost()` **outside** `Scaffold` on every screen that should show interstitials:

```kotlin
import io.surendrasp.ads.ui.HouseAdFullScreenHost

@Composable
fun SomeScreen() {
    Scaffold { paddingValues ->
        // ... screen content ...
    }
    HouseAdFullScreenHost()   // MUST be outside Scaffold
}
```

Then call `AdsSDK.onActionCompleted(context)` after every qualifying action:

```kotlin
import io.surendrasp.ads.core.AdsSDK

val context = LocalContext.current

Button(onClick = {
    performSave()
    AdsSDK.onActionCompleted(context)
}) { Text("Save") }
```

An interstitial fires on every `interstitialTriggerThreshold`-th call (default 3). The counter resets after firing.

### Step 6 — Add in-feed content ads (optional)

```kotlin
import io.surendrasp.ads.core.AdsSDK
import io.surendrasp.ads.core.AdListItem
import io.surendrasp.ads.ui.HouseAdCard

@Composable
fun ContentGrid(items: List<MyItem>) {
    val context = LocalContext.current
    val mixedList = remember(items) { AdsSDK.buildAdList(items, context) }

    LazyVerticalGrid(columns = GridCells.Fixed(3)) {
        items(mixedList) { listItem ->
            when (listItem) {
                is AdListItem.Content -> MyItemCard(listItem.item)
                is AdListItem.HouseAd -> HouseAdCard(listItem.config)
            }
        }
    }
}
```

---

## Complete public API

### `AdsSDK` (object)

```kotlin
object AdsSDK {
    fun init(context: Context, config: AdConfig)
    fun onActionCompleted(context: Context)
    fun <T> buildAdList(items: List<T>, context: Context): List<AdListItem<T>>
    fun nextHouseAd(context: Context): HouseAdConfig?

    val showFanBanner: StateFlow<Boolean>
    val showHouseAdFullScreen: SharedFlow<HouseAdConfig>
    val interstitialCloseDurationSec: StateFlow<Int>
    val theme: AdTheme
}
```

| Member | Description |
|--------|-------------|
| `init(context, config)` | Initialize. Call once in `Application.onCreate()`. Idempotent. |
| `onActionCompleted(context)` | Increment action counter; trigger interstitial at threshold. |
| `buildAdList(items, context)` | Return mixed list with house-ad placeholders injected every `contentAdInterval` items. |
| `nextHouseAd(context)` | Next uninstalled house ad in round-robin rotation, or `null` if all installed. |
| `showFanBanner` | `true` when FAN banner should show. Managed internally by `AdBannerSlot`. |
| `showHouseAdFullScreen` | Emits `HouseAdConfig` when a full-screen interstitial should show. Consumed by `HouseAdFullScreenHost`. |
| `interstitialCloseDurationSec` | Live close-button delay in seconds. Updated from RC after fetch. |
| `theme` | The `AdTheme` from `AdConfig`. Available after `init()`. |

---

### `AdConfig` (data class)

```kotlin
data class AdConfig(
    val fanBannerPlacementId:        String,                // required
    val fanInterstitialPlacementId:  String,                // required
    val analytics:                   FirebaseAnalytics,     // required
    val remoteConfig:                FirebaseRemoteConfig,  // required
    val appPackageId:                String,                // required — use BuildConfig.APPLICATION_ID
    val theme:                       AdTheme = AdTheme(),
)
```

All cadence and timing values (`bannerHouseCadence`, `interstitialHouseCadence`, `contentAdInterval`, `interstitialTriggerThreshold`, `interstitialCloseDurationSec`) are configured via the `ads_cadence_config` Firebase Remote Config key. SDK built-in defaults:

| Setting | Default |
|---------|---------|
| `bannerHouseCadence` | 50 |
| `interstitialHouseCadence` | 10 |
| `contentAdInterval` | 4 |
| `interstitialTriggerThreshold` | 3 |
| `interstitialCloseSec` | 5 |

**Cadence semantics:**

| Value | Banner / Interstitial behaviour | Content-ad behaviour |
|-------|---------------------------------|----------------------|
| `-1` | House ads disabled; FAN only | No ads injected |
| `0` | Always show house ad; FAN never used | No ads injected (treated same as `-1`) |
| `N > 0` | 1 house ad per N FAN impressions | 1 house ad per N content items |

---

### `AdTheme` (data class)

```kotlin
data class AdTheme(
    val primary:   Color? = null,
    val onPrimary: Color? = null,
    val surface:   Color? = null,
    val onSurface: Color? = null,
    val cardShape: Shape? = null,
    val ctaShape:  Shape? = null,
) {
    companion object {
        val DEFAULT_CARD_SHAPE: Shape = RoundedCornerShape(12.dp)
        val DEFAULT_CTA_SHAPE:  Shape = RoundedCornerShape(8.dp)
    }
}
```

| Field | Used for | Fallback when `null` |
|-------|---------|----------------------|
| `primary` | CTA button background; icon placeholder tint | `MaterialTheme.colorScheme.primary` |
| `onPrimary` | CTA button text / icon | `MaterialTheme.colorScheme.onPrimary` |
| `surface` | Banner row background; card gradient | `MaterialTheme.colorScheme.secondaryContainer` |
| `onSurface` | Title, subtitle, "Ad" badge text | `MaterialTheme.colorScheme.onSecondaryContainer` |
| `cardShape` | `HouseAdCard` clip shape | `RoundedCornerShape(12.dp)` |
| `ctaShape` | CTA button shape (all composables) | `RoundedCornerShape(8.dp)` |

---

### `HouseAdConfig` (data class)

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
) {
    fun resolvedTitle(context: Context): String
    fun resolvedSubtitle(context: Context): String
    fun resolvedCtaText(context: Context): String
}
```

Deserialized from the `house_ads` RC key. Passed to slot composables and `AdListItem.HouseAd`. `priority` is assigned from the JSON array index (index 0 = highest priority = shown first).

**Always call `resolved*` methods when rendering text** — they return the Hindi string automatically when the device/app locale is `hi`, falling back to the English field. Do not access `title`, `subtitle`, or `ctaText` directly in custom UI.

Locale resolution order:
1. App-level locale set via `AppCompatDelegate.setApplicationLocales`
2. Device system language (`Locale.getDefault().language`)
3. English fallback

---

### `AdListItem<T>` (sealed interface)

```kotlin
sealed interface AdListItem<out T> {
    data class Content<T>(val item: T)            : AdListItem<T>
    data class HouseAd(val config: HouseAdConfig) : AdListItem<Nothing>
}
```

Returned by `AdsSDK.buildAdList()`. Exhaustive `when` required.

---

### Composables

#### `AdBannerSlot`

```kotlin
@Composable
fun AdBannerSlot(
    modifier:       Modifier = Modifier,
    houseAdContent: (@Composable (config: HouseAdConfig) -> Unit)? = null,
)
```

Import: `io.surendrasp.ads.ui.AdBannerSlot`

Renders FAN banner or house-ad banner. The FAN `AdView` is created once and visibility-toggled to prevent recomposition flicker. `houseAdContent` slot replaces the SDK default `HouseAdBanner`.

---

#### `HouseAdFullScreenHost`

```kotlin
@Composable
fun HouseAdFullScreenHost(
    content: (@Composable (config: HouseAdConfig, onDismiss: () -> Unit) -> Unit)? = null,
)
```

Import: `io.surendrasp.ads.ui.HouseAdFullScreenHost`

Must be placed **outside** `Scaffold`. Collects `AdsSDK.showHouseAdFullScreen` and renders the overlay. `content` slot replaces the SDK default `HouseAdFullScreen`. **When using the slot, you must call `onDismiss` when the user closes.**

---

#### `HouseAdBanner`

```kotlin
@Composable
fun HouseAdBanner(
    config:   HouseAdConfig,
    modifier: Modifier = Modifier,
    onShown:  () -> Unit = {},
)
```

Import: `io.surendrasp.ads.ui.HouseAdBanner`

50 dp tall. Use directly only if building fully custom banner UI outside `AdBannerSlot`.

---

#### `HouseAdCard`

```kotlin
@Composable
fun HouseAdCard(
    config:   HouseAdConfig,
    modifier: Modifier = Modifier,
)
```

Import: `io.surendrasp.ads.ui.HouseAdCard`

Square (1:1) house-ad card for grids. Entire surface is tappable and opens Play Store. Override `modifier` for non-square layouts.

---

#### `HouseAdFullScreen`

```kotlin
@Composable
fun HouseAdFullScreen(
    config:           HouseAdConfig,
    closeDurationSec: Int,
    onDismiss:        () -> Unit,
)
```

Import: `io.surendrasp.ads.ui.HouseAdFullScreen`

Full-screen interstitial overlay. Normally used only internally via `HouseAdFullScreenHost`. Use directly if you need lower-level control.

---

## Firebase Remote Config keys

### `house_ads`

JSON array. List of cross-sell apps to promote.

```json
[
  {
    "packageName":  "com.example.otherapp",
    "title":        "My Other App",
    "titleHi":      "मेरा दूसरा ऐप",
    "subtitle":     "A great tool for everyone",
    "subtitleHi":   "सबके लिए एक बेहतरीन टूल",
    "iconUrl":      "https://example.com/icon.png",
    "playStoreUrl": "https://play.google.com/store/apps/details?id=com.example.otherapp",
    "ctaText":      "Install Free",
    "ctaTextHi":    "मुफ्त इंस्टॉल करें"
  }
]
```

Required fields: `packageName`, `title`, `playStoreUrl`. All Hindi fields (`titleHi`, `subtitleHi`, `ctaTextHi`) are optional — omit them to show English only.

Apps that are already installed on the device are automatically skipped — the SDK checks `PackageManager` per call.

Banner and interstitial slots cycle through all eligible apps in **round-robin** order (not always highest-priority). Each call to `nextHouseAd()` / `nextAd()` advances the internal index and wraps around.

### `ads_cadence_config`

JSON object. All fields optional. Overrides SDK built-in defaults at runtime.

```json
{
  "interstitialCloseSec":         5,
  "bannerHouseCadence":           50,
  "interstitialHouseCadence":     10,
  "contentAdInterval":            4,
  "interstitialTriggerThreshold": 3
}
```

| Field | SDK default | Description |
|-------|-------------|-------------|
| `interstitialCloseSec` | `5` | Seconds before close button appears; drives countdown ring; exposed as `AdsSDK.interstitialCloseDurationSec` |
| `bannerHouseCadence` | `50` | House-ad cadence for banner slot |
| `interstitialHouseCadence` | `10` | House-ad cadence for interstitial slot |
| `contentAdInterval` | `4` | Inject 1 house-ad card per N real content items |
| `interstitialTriggerThreshold` | `3` | `onActionCompleted()` calls before an interstitial fires |

Changes take effect on the next impression after RC fetch completes (no restart required).

---

## Analytics events

All events are sent to the host app's `FirebaseAnalytics`. All events include `app_package_id` (from `AdConfig.appPackageId`).

| Event | Extra parameters | Meaning |
|-------|-----------------|---------|
| `ad_impression` | `slot`, `source` | Any ad became visible |
| `ad_click` | `slot`, `source` | User tapped the ad CTA |
| `ad_load_failed` | `slot`, `error_code`, `error_message` | FAN banner failed to load |
| `house_ad_skipped` | `slot` | House ad was eligible but skipped (no uninstalled app found) |
| `interstitial_shown` | `source` (`"fan"` \| `"house"` \| `"house_fallback"`) | Interstitial became visible |
| `interstitial_dismissed` | `source`, `visible_seconds` | User tapped X to close house-ad interstitial |
| `interstitial_timed_close` | `source`, `visible_seconds` | Countdown expired; close button now shown |
| `interstitial_app_drop` | `source`, `visible_seconds` | User backgrounded app while interstitial was on screen |

---

## Slot API — custom UI

All three ad surfaces accept a composable slot for fully custom UI. The SDK retains ownership of timing, analytics, and show/dismiss decisions.

### Custom banner

```kotlin
AdBannerSlot(
    houseAdContent = { config: HouseAdConfig ->
        // Use config.resolvedTitle(context), config.resolvedSubtitle(context),
        // config.resolvedCtaText(context) for locale-aware strings.
        // config also has: iconUrl, playStoreUrl, packageName
        // Recommended height: 50.dp to match FAN banner
        MyBannerView(config)
    }
)
```

### Custom interstitial

```kotlin
HouseAdFullScreenHost(
    content = { config: HouseAdConfig, onDismiss: () -> Unit ->
        // You MUST call onDismiss when user closes; SDK fires analytics on your call
        MyInterstitialView(config = config, onClose = onDismiss)
    }
)
```

### Custom content card

Do not call `HouseAdCard`; render your own composable for `AdListItem.HouseAd`:

```kotlin
is AdListItem.HouseAd -> MyCustomAdCard(item.config)
```

---

## Interstitial trigger logic (full decision tree)

When `AdsSDK.onActionCompleted(context)` is called:

1. Increment internal action counter.
2. If `actionCount % interstitialTriggerThreshold != 0` → stop (not at threshold yet).
3. Reset action counter.
4. Evaluate house-ad interstitial turn (`fanInterstitialCount % interstitialHouseCadence == 0`):
   - If it's a house turn AND `interstitialHouseCadence != -1` AND an eligible house ad exists → emit to `showHouseAdFullScreen`, preload FAN for next time, log `interstitial_shown(source="house")`, return.
   - If all apps installed → log `house_ad_skipped`, fall through to FAN.
5. If `fanInterstitial.isReady` → show FAN interstitial, log `interstitial_shown(source="fan")`.
6. FAN not ready AND `interstitialHouseCadence != -1` AND eligible house ad exists → emit to `showHouseAdFullScreen`, log `interstitial_shown(source="house_fallback")`.
7. Nothing shown → silent skip.

---

## Banner display logic

`AdBannerSlot` observes `AdsSDK.showFanBanner` (a `StateFlow<Boolean>`):

- Starts `true` by default.
- **When FAN banner loads successfully:**
  - If `bannerHouseCadence == 0` (always house) → check for eligible house ad; if found, set `showFanBanner = false`; if none, stay on FAN.
  - Otherwise → count the impression; if it's the house-ad turn, switch to house banner.
- **When FAN banner fails to load:**
  - If `bannerHouseCadence == -1` → show nothing.
  - Else if house ad available → switch to house banner.
  - Else → hide banner entirely (`showFanBanner = false`).
- **When house banner is shown:**
  - If `bannerHouseCadence == 0` → stay on house banner (don't reset to FAN).
  - Otherwise → reset `showFanBanner = true` so FAN tries again next time.

---

## Content ad injection

`AdsSDK.buildAdList(items, context)` works as follows:

- If `contentAdInterval <= 0` (i.e., `-1` or `0`) or `items` is empty → return `items` unchanged as `AdListItem.Content`. Note: `0` is treated as disabled, not "always inject".
- Build eligible queue: all uninstalled apps in priority order.
- If queue is empty → return `items` unchanged.
- Iterate `items`; after every `contentAdInterval` items, pop one ad from the queue and append `AdListItem.HouseAd`.
- When queue is exhausted, remaining injection points are silently skipped (no gap).

---

## FAN initialization notes

FAN (`AudienceNetworkAds.initialize`) runs on a background IO thread in `GlobalScope` to avoid blocking the main thread at app startup (it does disk I/O and can cost ~50 ms).

The FAN `AdView` for banners is created inside the `AdBannerSlot` composable via `AndroidView`, because `AdView` requires a `Context` only available at composition time.

FAN interstitials are preloaded after each dismissal so a fresh ad is ready for the next trigger.

---

## Integration checklist

Use this checklist to verify a complete integration:

### Repository & dependency setup
- [ ] `settings.gradle.kts` has the `SmartAdsSDK` Maven repository block with credential providers
- [ ] `build.gradle.kts` has `implementation("com.github.SurendraSp:smart-ads:1.0.4")`
- [ ] `build.gradle.kts` has `io.coil-kt:coil-compose:2.x`
- [ ] `build.gradle.kts` has Firebase BOM + `firebase-analytics` + `firebase-config`
- [ ] `com.facebook.android:audience-network-sdk` is **not** added manually (it's bundled)
- [ ] Credentials available via `local.properties`, env vars, or `gh auth login`

### Application class
- [ ] `Application` subclass exists
- [ ] `AdsSDK.init(context, config)` called in `Application.onCreate()`
- [ ] `fanBannerPlacementId` set to a real FAN placement ID
- [ ] `fanInterstitialPlacementId` set to a real FAN placement ID
- [ ] `analytics` set to `FirebaseAnalytics.getInstance(this)`
- [ ] `remoteConfig` set to `Firebase.remoteConfig`
- [ ] `appPackageId` set to `BuildConfig.APPLICATION_ID`
- [ ] Application registered in `AndroidManifest.xml` via `android:name=".MyApp"`

### Banner
- [ ] `AdBannerSlot()` placed in at least one screen
- [ ] Import: `io.surendrasp.ads.ui.AdBannerSlot`
- [ ] `modifier = Modifier.fillMaxWidth()` applied (or appropriate size)

### Interstitial
- [ ] `HouseAdFullScreenHost()` placed **outside** `Scaffold` on screens that show interstitials
- [ ] Import: `io.surendrasp.ads.ui.HouseAdFullScreenHost`
- [ ] `AdsSDK.onActionCompleted(context)` called after save / download / share / export actions
- [ ] Import: `io.surendrasp.ads.core.AdsSDK`

### In-feed ads (if applicable)
- [ ] `AdsSDK.buildAdList(items, context)` called to produce the mixed list
- [ ] `AdListItem.Content` and `AdListItem.HouseAd` handled in `when`
- [ ] Import: `io.surendrasp.ads.core.AdListItem`
- [ ] `HouseAdCard` (or custom composable) rendered for `AdListItem.HouseAd`

### Firebase Remote Config
- [ ] `house_ads` RC key configured with a JSON array of at least one house ad
- [ ] Each house-ad entry has `packageName`, `title`, `playStoreUrl`
- [ ] `ads_cadence_config` RC key configured (or left absent to use SDK built-in defaults)
- [ ] RC minimum fetch interval set to 0 for debug builds (optional but recommended for testing)

### Theming (optional)
- [ ] `AdTheme` passed in `AdConfig` with brand colors
- [ ] `primary` / `onPrimary` set for CTA button
- [ ] `surface` / `onSurface` set for banner/card backgrounds

### Slot API (if custom UI used)
- [ ] `houseAdContent` lambda provided to `AdBannerSlot` (if custom banner)
- [ ] `content` lambda provided to `HouseAdFullScreenHost` (if custom interstitial)
- [ ] `onDismiss` called inside custom interstitial when user closes

### Testing
- [ ] FAN placement IDs are test IDs during development (use Meta's test IDs)
- [ ] At least one entry in `house_ads` RC is an app **not** installed on test device (otherwise no house ads show)
- [ ] `onActionCompleted` called enough times (`interstitialTriggerThreshold` times) to trigger interstitial
- [ ] `HouseAdFullScreenHost` is reachable (not conditionally hidden) when testing interstitials
- [ ] Firebase Remote Config fetch verified (check logcat for `ads_cadence_config applied`)

---

## Common mistakes

| Mistake | Fix |
|---------|-----|
| `AdsSDK not initialized` crash | `AdsSDK.init()` not called before `AdsSDK.onActionCompleted()` or composable composition. Ensure `init()` is in `Application.onCreate()` and the Application is registered in the manifest. |
| No house ads shown | All apps in `house_ads` are installed on the test device. Add an app that is not installed. |
| Banner not visible | `showFanBanner` may be `false` because FAN failed and no house ad is eligible. Check that `house_ads` RC key has a valid entry. |
| Interstitial never fires | `onActionCompleted()` not called, or called fewer than `interstitialTriggerThreshold` times. Or `HouseAdFullScreenHost` is inside `Scaffold` (overlay won't render correctly). |
| Duplicate class error at build | `firebase-analytics` or `firebase-config` added as `implementation` without BOM, conflicting with a version already in the project. Use Firebase BOM. |
| Missing Coil crash | `coil-compose` not added to the app's dependencies. SDK declares it `compileOnly`. |
| `422 Unprocessable Entity` on publish | Only relevant if publishing a new SDK version. Bump `VERSION_NAME` in `gradle.properties` — versions cannot be overwritten on GitHub Packages. |
| RC changes not reflected | Default RC fetch interval is 1 hour. Set `minimumFetchIntervalInSeconds = 0` in debug builds. |
