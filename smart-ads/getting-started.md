# Smart Ads SDK — Getting Started

**Maven:** `com.github.SurendraSp:smart-ads:1.0.6-RC`  
**Registry:** `https://maven.pkg.github.com/SurendraSp/SmartAdsSDK`  
**Namespace:** `io.surendrasp.ads`

---

## Requirements

| Requirement | Version |
|-------------|---------|
| Android `minSdk` | 26 |
| `compileSdk` | 37 |
| JDK | 21 |
| Kotlin | 2.x |
| Jetpack Compose BOM | 2025.x |
| Facebook Audience Network | 6.21.0 |
| Firebase Analytics | via BOM 34.x |
| Firebase Remote Config | via BOM 34.x |
| Coil Compose | 2.x *(host app must provide)* |

---

## 1. Add the Maven repository

The SDK is published to GitHub Packages. Authentication is required even for read access.

In `settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
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
    }
}
```

### Credential options (pick one)

**Option A — GitHub CLI** *(recommended; no extra config if already logged in)*

```bash
gh auth login
```

The SDK's Gradle build will call `gh auth token` automatically as a fallback. No `local.properties` needed.

**Option B — `local.properties`** *(never commit this file)*

```properties
gpr.user=your-github-username
gpr.key=ghp_xxxxxxxxxxxxxxxxxxxx
```

The PAT needs `read:packages` scope. Generate one at GitHub → Settings → Developer settings → Personal access tokens.

**Option C — environment variables** *(CI / scripts)*

```bash
export GPR_USER=your-github-username
export SMART_ADS_SDK_GPR_KEY=ghp_xxxxxxxxxxxxxxxxxxxx
```

In `build.gradle.kts` for CI:

```yaml
- name: Build
  run: ./gradlew assembleRelease
  env:
    GPR_USER: ${{ github.actor }}
    SMART_ADS_SDK_GPR_KEY: ${{ secrets.SMART_ADS_SDK_GPR_KEY }}
```

---

## 2. Add the dependency

In your app's `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.github.SurendraSp:smart-ads:1.0.6-RC")

    // ── Required peer dependencies ────────────────────────────────────────────
    // The SDK declares these compileOnly to keep its AAR lean and let you control
    // versions. If any of these are missing at runtime the app will crash with
    // ClassNotFoundException. Facebook Audience Network is the only bundled dep —
    // you do NOT need to add it separately.

    // Coil 3 — SDK uses AsyncImage for house-ad icons
    implementation("io.coil-kt.coil3:coil-compose:3.5.0")
    implementation("io.coil-kt.coil3:coil-network-okhttp:3.5.0")  // required for URL image loading

    // Firebase
    implementation(platform("com.google.firebase:firebase-bom:34.14.1"))
    implementation("com.google.firebase:firebase-analytics")
    implementation("com.google.firebase:firebase-config")
}
```

> **Minimum versions.** The SDK is compiled against the versions above. Your app may use higher versions — Gradle's highest-wins rule ensures compatibility. Using a *lower* version than listed risks a `NoSuchMethodException` at runtime.

> **Facebook Audience Network** is bundled inside the SDK AAR — do not add it to your own `build.gradle`.

---

## 3. Initialize in Application.onCreate

Create or update your `Application` subclass:

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()

        AdsSDK.init(
            context = applicationContext,
            config  = AdConfig(
                // FAN placement IDs from Meta Business Manager
                fanBannerPlacementId       = "YOUR_BANNER_PLACEMENT_ID",
                fanInterstitialPlacementId = "YOUR_INTERSTITIAL_PLACEMENT_ID",

                analytics    = FirebaseAnalytics.getInstance(this),
                remoteConfig = Firebase.remoteConfig,
                appPackageId = BuildConfig.APPLICATION_ID,

                theme = AdTheme(
                    primary   = Color(0xFF1B8C3E),
                    onPrimary = Color.White,
                    surface   = Color(0xFFE8F5E9),
                    onSurface = Color(0xFF1B5E20),
                    cardShape = RoundedCornerShape(16.dp),
                    ctaShape  = RoundedCornerShape(50.dp),
                ),
            )
        )
    }
}
```

> **Cadence and timing** (how often ads show, how long the close countdown is, how aggressively to ramp ads for new users) are controlled via Firebase Remote Config key `ads_cadence_config` — no code changes needed to tune them. See [Remote Config](remote-config.md) for the full schema, the progressive-cadence ramp behaviour, and SDK built-in defaults.

> **House ads are opt-in.** Without both `house_ads` and `ads_cadence_config` Remote Config entries, the SDK runs in FAN-only mode automatically. See the [FAN-only fallback](remote-config.md#fan-only-fallback-106-rc-and-later) section.

Register `MyApp` in `AndroidManifest.xml`:

```xml
<application
    android:name=".MyApp"
    ...>
```

`AdsSDK.init()` is safe to call multiple times — subsequent calls after the first are no-ops.

---

## 4. Place UI composables

### Banner

Drop `AdBannerSlot()` anywhere in a screen. Typically at the bottom:

```kotlin
@Composable
fun HomeScreen() {
    Column {
        // ... your screen content
        AdBannerSlot(modifier = Modifier.fillMaxWidth())
    }
}
```

### Interstitial

Place `HouseAdFullScreenHost()` **once** at the root of each screen that shows interstitials, outside `Scaffold` so it overlays everything:

```kotlin
@Composable
fun MainScreen() {
    Scaffold { paddingValues ->
        // ... your screen content
    }
    HouseAdFullScreenHost()   // must be outside Scaffold
}
```

Then call `AdsSDK.onActionCompleted(context)` after qualifying user actions (save, download, share, export, etc.):

```kotlin
Button(onClick = {
    viewModel.saveItem()
    AdsSDK.onActionCompleted(context)
}) {
    Text("Save")
}
```

An interstitial fires after every `interstitialTriggerThreshold` calls. The *effective* threshold ramps from `interstitialTriggerThresholdStart` (gentle, for new users) down to `interstitialTriggerThreshold` (the steady-state target) as the user racks up sessions and actions. See [Remote Config — How the ramp works](remote-config.md#how-the-ramp-works).

### Content / In-feed ads

```kotlin
@Composable
fun GalleryScreen(viewModel: GalleryViewModel) {
    val items   by viewModel.items.collectAsState()
    val context = LocalContext.current

    // Inject house-ad placeholders at the effective contentAdInterval.
    // Key on AdsSDK.houseAds so the list rebuilds when RC fetch lands —
    // without this the first composition caches a no-ads result forever.
    val houseAds  by AdsSDK.houseAds.collectAsStateWithLifecycle()
    val mixedList = remember(items, houseAds) { AdsSDK.buildAdList(items, context) }

    LazyVerticalGrid(columns = GridCells.Fixed(3)) {
        items(mixedList) { item ->
            when (item) {
                is AdListItem.Content -> ContentCard(item.item)
                is AdListItem.HouseAd -> HouseAdCard(item.config)
            }
        }
    }
}
```

---

## 5. AdConfig parameter reference

| Parameter | Type | Description |
|-----------|------|-------------|
| `fanBannerPlacementId` | `String` | FAN banner placement ID |
| `fanInterstitialPlacementId` | `String` | FAN interstitial placement ID |
| `analytics` | `FirebaseAnalytics` | Host app's Firebase Analytics instance |
| `remoteConfig` | `FirebaseRemoteConfig` | Host app's Firebase Remote Config instance |
| `appPackageId` | `String` | Host app package name (tagged on all events) |
| `theme` | `AdTheme` | Optional brand theming; see [Theming](theming.md) |

All cadence and timing values are controlled via RC — see [Remote Config](remote-config.md).
