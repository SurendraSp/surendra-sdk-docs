# Smart Ads Mediation SDK — Getting Started

**Maven:** `io.surendrasp:ads-sdk-mediation:1.0.0-RC1`  
**Registry:** `https://maven.pkg.github.com/SurendraSp/SmartAdsSDK`  
**Namespace:** `io.surendrasp.ads.mediation`

---

## Requirements

| Requirement | Version |
|-------------|---------|
| Android `minSdk` | 26 |
| `compileSdk` | 37 |
| JDK | 21 |
| Kotlin | 2.x |
| Jetpack Compose BOM | 2025.x |
| Google Mobile Ads (GMA) | 25.4.0 *(pulled in via POM — no explicit dep needed)* |
| Firebase Analytics | via BOM 34.x |
| Firebase Remote Config | via BOM 34.x |
| Coil Compose | 3.5.0 *(host app must provide)* |
| Coil Network OkHttp | 3.5.0 *(host app must provide)* |

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

**Option A — GitHub CLI** *(recommended)*

```bash
gh auth login
```

The SDK's Gradle build calls `gh auth token` automatically as a fallback — no `local.properties` needed.

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

---

## 2. Add the dependency

In your app's `build.gradle.kts`:

```kotlin
dependencies {
    // Core mediation SDK (ads-sdk-kit is pulled in transitively)
    implementation("io.surendrasp:ads-sdk-mediation:1.0.0-RC1")

    // ── Required peer dependencies ────────────────────────────────────────────
    // The SDK declares these compileOnly to keep the AAR lean.
    // Missing any of these at runtime → ClassNotFoundException crash.

    // Coil 3 — SDK uses AsyncImage for house-ad icons
    implementation("io.coil-kt.coil3:coil-compose:3.5.0")
    implementation("io.coil-kt.coil3:coil-network-okhttp:3.5.0")

    // Firebase
    implementation(platform("com.google.firebase:firebase-bom:34.15.0"))
    implementation("com.google.firebase:firebase-analytics")
    implementation("com.google.firebase:firebase-config")

    // ── Optional: AdMob mediation adapters ───────────────────────────────────
    // Add ONLY the networks you have configured in the AdMob dashboard.
    // The SDK loads adapters via reflection — missing adapters are skipped silently.

    // Meta (Facebook) Audience Network adapter
    implementation("com.google.ads.mediation:facebook:6.21.0.3")

    // InMobi adapter
    implementation("com.google.ads.mediation:inmobi:11.3.0.0")
}
```

> **GMA is bundled.** `play-services-ads:25.4.0` is declared as a runtime dependency in the SDK POM — Gradle pulls it in automatically. Do not add it separately unless you need to pin a different version.

> **`ads-sdk-kit` is transitively included.** You do not need to declare it explicitly. Gradle resolves it from the POM.

---

## 3. Add the AdMob App ID to your manifest

AdMob requires your App ID in `AndroidManifest.xml`. Without this the app crashes at launch with `"The Google Mobile Ads SDK was initialized incorrectly."`:

```xml
<manifest>
    <application>
        <meta-data
            android:name="com.google.android.gms.ads.APPLICATION_ID"
            android:value="ca-app-pub-XXXXXXXXXXXXXXXX~XXXXXXXXXX" />
    </application>
</manifest>
```

Replace `ca-app-pub-XXXXXXXXXXXXXXXX~XXXXXXXXXX` with your real AdMob App ID from the AdMob console.

---

## 4. Initialize in Application.onCreate

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()

        AdsSDK.init(
            context = applicationContext,
            config  = MediationConfig(
                // AdMob ad unit IDs from the AdMob console
                admobBannerAdUnitId       = "ca-app-pub-XXXXXXXXXXXXXXXX/XXXXXXXXXX",
                admobInterstitialAdUnitId = "ca-app-pub-XXXXXXXXXXXXXXXX/XXXXXXXXXX",

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

> **Test ad unit IDs.** During development use [Google's official test ad unit IDs](https://developers.google.com/admob/android/test-ads#sample_ad_units) instead of your real IDs. Using real IDs during testing risks account suspension.

Register `MyApp` in `AndroidManifest.xml`:

```xml
<application
    android:name=".MyApp"
    ...>
```

`AdsSDK.init()` is safe to call multiple times — subsequent calls are no-ops.

---

## 5. Place UI composables

### Banner

Drop `AdBannerSlot()` anywhere in a screen. It renders an adaptive full-width banner that matches the screen width and adjusts its height automatically:

```kotlin
@Composable
fun HomeScreen() {
    Column {
        // ... your screen content
        AdBannerSlot(modifier = Modifier.fillMaxWidth())
    }
}
```

The banner width is driven by `AdSize.getCurrentOrientationAnchoredAdaptiveBannerAdSize`, so it fills the screen on both portrait and landscape and handles density-independent dimensions correctly.

### Interstitial

Place `HouseAdFullScreenHost()` **once** at the root of each screen, outside `Scaffold`:

```kotlin
@Composable
fun MainScreen() {
    Scaffold { paddingValues ->
        // ... your screen content
    }
    HouseAdFullScreenHost()   // must be outside Scaffold
}
```

Then call `AdsSDK.onActionCompleted(context)` after qualifying user actions:

```kotlin
Button(onClick = {
    viewModel.saveItem()
    AdsSDK.onActionCompleted(context)
}) {
    Text("Save")
}
```

The SDK decides whether to show a house ad (from Remote Config) or an AdMob interstitial based on the current cadence counter. Cadence is controlled entirely via Firebase Remote Config — see [Remote Config](remote-config.md).

### Content / In-feed ads

```kotlin
@Composable
fun GalleryScreen(viewModel: GalleryViewModel) {
    val items   by viewModel.items.collectAsState()
    val context = LocalContext.current

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

## 6. MediationConfig parameter reference

| Parameter | Type | Description |
|-----------|------|-------------|
| `admobBannerAdUnitId` | `String` | AdMob banner ad unit ID |
| `admobInterstitialAdUnitId` | `String` | AdMob interstitial ad unit ID |
| `analytics` | `FirebaseAnalytics` | Host app's Firebase Analytics instance |
| `remoteConfig` | `FirebaseRemoteConfig` | Host app's Firebase Remote Config instance |
| `appPackageId` | `String` | Host app package name (tagged on all events) |
| `theme` | `AdTheme` | Optional brand theming; see [Theming](../smart-ads/theming.md) |

All cadence and timing values are controlled via Firebase Remote Config — see [Remote Config](remote-config.md).

---

## 7. Mediation network setup

To show ads from Facebook Audience Network or InMobi, you must:

1. Add the adapter dependency (step 2 above).
2. Configure the network in your AdMob dashboard under **Mediation → Create mediation group**.
3. Add the network account credentials in AdMob (FAN App ID + Placement IDs, InMobi Account ID + Placement IDs).

AdMob handles the waterfall routing — you do not write any adapter code in the SDK or in your app. See [Mediation Setup](mediation-setup.md) for a step-by-step walkthrough of the AdMob dashboard configuration.
