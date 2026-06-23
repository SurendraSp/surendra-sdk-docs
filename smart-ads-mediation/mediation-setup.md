# Smart Ads Mediation SDK — Mediation Network Setup

This guide walks through configuring mediation networks in the AdMob dashboard so your AdMob ad units serve ads from Facebook Audience Network (FAN) and InMobi in addition to Google's own demand.

---

## How AdMob mediation works

AdMob acts as the waterfall controller. When your app requests an ad:

1. AdMob contacts each mediation network in priority order.
2. The first network that can fill the request returns the ad.
3. Your app displays it via the SDK's composable — no per-network code in your app.

You configure the waterfall in the AdMob dashboard. Adding adapter dependencies to your `build.gradle.kts` (step 2 in [Getting Started](getting-started.md)) is all that's needed in code.

---

## Prerequisites

- An AdMob account at [admob.google.com](https://admob.google.com)
- Your app created in AdMob and the App ID added to `AndroidManifest.xml`
- At least one ad unit created for each format (banner, interstitial)
- Accounts on the ad networks you want to add (Meta Business Manager for FAN, InMobi dashboard for InMobi)

---

## Step 1: Create a mediation group

1. Open the AdMob console → **Mediation** → **Create mediation group**.
2. **Ad format:** choose *Banner* or *Interstitial* (repeat for each format).
3. **Platform:** *Android*.
4. **Name:** choose a descriptive name (e.g., `Android Banner - All Countries`).
5. **Status:** *Enabled*.
6. Click **Continue**.

---

## Step 2: Add your AdMob ad unit

Under **Ad units**, click **Add ad units** and select the AdMob ad unit created for this format. This is the same ad unit ID you passed to `MediationConfig.admobBannerAdUnitId` or `admobInterstitialAdUnitId`.

---

## Step 3: Add mediation network sources

Under **Ad sources**, click **Add custom event** or **Add ad network**.

### Meta (Facebook) Audience Network

1. Click **Add ad network** → select **Meta Audience Network**.
2. Enter your **FAN App ID** (from Meta Business Manager → Apps).
3. Click **Done**, then **Continue** to the placement IDs screen.
4. Enter the **Placement ID** for each ad unit:
   - For banners: select `320x50` or `responsive` format.
   - For interstitials: select `Interstitial` format.
5. Set the **eCPM floor** (or leave at `0` to let AdMob optimize).

> **FAN test placements.** During testing, use Meta's test placement IDs instead of live ones. Live FAN placements with low fill can push AdMob to fill the request itself.

### InMobi

1. Click **Add ad network** → select **InMobi**.
2. Enter your **InMobi Account ID** (from the InMobi dashboard → Account → API).
3. Enter the **Placement ID** for each ad unit.
4. Set the eCPM floor.

---

## Step 4: Adaptive banner in AdMob

For banners, the SDK uses `AdSize.getCurrentOrientationAnchoredAdaptiveBannerAdSize` — an adaptive size that covers the full screen width. In AdMob:

- When creating the banner ad unit, select **Adaptive banner** as the ad size.
- No special setting is needed for mediation networks — FAN and InMobi adapters return creatives; AdMob scales them to fit the requested size.

---

## Step 5: Test devices

Register your test device in AdMob to receive test ads without risking policy violations:

1. AdMob console → **Settings** → **Test devices** → **Add test device**.
2. Enter your device's advertising ID (find it in Settings → Google → Ads → Advertising ID on the device).
3. AdMob returns test ads to this device regardless of live fill.

Alternatively, use [Google's official test ad unit IDs](https://developers.google.com/admob/android/test-ads#sample_ad_units) during development — no test-device registration required.

---

## Step 6: Verify adapter loading

Run the app in debug mode and filter logcat by `Ads`:

```
Ads: MobileAds initialized
Ads: Mediation adapter loaded: com.google.ads.mediation.facebook.FacebookMediationAdapter
Ads: Mediation adapter loaded: com.google.ads.mediation.inmobi.InMobiMediationAdapter
```

If an adapter is missing from the log, the adapter dependency is either not added to `build.gradle.kts` or its version does not match the one expected by GMA 25.4.0.

---

## Supported adapter versions (GMA 25.4.0)

| Network | Adapter artifact | Version |
|---------|-----------------|---------|
| Meta (FAN) | `com.google.ads.mediation:facebook` | `6.21.0.3` |
| InMobi | `com.google.ads.mediation:inmobi` | `11.3.0.0` |

Always check [Google's mediation adapter changelog](https://developers.google.com/admob/android/mediation) for the latest compatible versions before upgrading GMA.
