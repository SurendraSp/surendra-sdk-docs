# Smart Ads SDK — Firebase Remote Config

The SDK reads two RC keys at startup and re-fetches every hour. All RC values are optional — the SDK falls back to `AdConfig` values, then to built-in defaults if a key is missing or malformed.

---

## RC keys

| Key | RC Type | Description |
|-----|---------|-------------|
| `house_ads` | JSON string (array) | List of cross-sell apps to advertise |
| `ads_cadence_config` | JSON string (object) | Cadence and timing overrides |

---

## `house_ads`

A JSON array of house-ad objects. Each object maps to one `HouseAdConfig`. The array is priority-ordered: index 0 is shown first among all uninstalled apps.

### Schema

```json
[
  {
    "packageName":  "com.example.otherapp",
    "title":        "My Other App",
    "subtitle":     "Short tagline shown under the title",
    "iconUrl":      "https://cdn.example.com/icon.png",
    "playStoreUrl": "https://play.google.com/store/apps/details?id=com.example.otherapp",
    "ctaText":      "Install Free"
  }
]
```

### Field reference

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `packageName` | string | yes | — | Android package name. Used to check if the app is already installed — installed apps are never shown. |
| `title` | string | yes | — | App name displayed in banner, card, and full-screen. |
| `subtitle` | string | no | `""` | Short description / tagline. |
| `iconUrl` | string | no | `""` | URL of the app icon. Loaded with Coil. Empty string → placeholder tinted with `AdTheme.primary`. |
| `playStoreUrl` | string | yes | — | Full Play Store URL. Opened via `Intent.ACTION_VIEW` when the user taps the CTA or full-screen background. |
| `ctaText` | string | no | `"Install"` | Label for the CTA button. |

> `priority` is not set in JSON — it is assigned automatically from the array index during parsing (index 0 → priority 0, highest).

### Priority logic

The `HouseAdEngine` iterates the list from index 0 and returns the first app whose `packageName` is **not installed** on the device:

- `nextAd()` — used for banner and interstitial; returns the single highest-priority eligible ad.
- `buildEligibleQueue()` — used for in-feed content ads; returns all eligible ads in priority order. The orchestrator cycles through this queue round-robin across content list injection points.

If every app in `house_ads` is already installed, no house ads are shown for any slot.

### Sample RC value

```json
[
  {
    "packageName":  "com.socialtool.videosaver",
    "title":        "Video Saver Pro",
    "subtitle":     "Download any video in seconds",
    "iconUrl":      "https://example.com/icons/videosaver.png",
    "playStoreUrl": "https://play.google.com/store/apps/details?id=com.socialtool.videosaver",
    "ctaText":      "Install Free"
  },
  {
    "packageName":  "com.socialtool.statussaver",
    "title":        "Status Saver",
    "subtitle":     "Save WhatsApp statuses in one tap",
    "iconUrl":      "https://example.com/icons/statussaver.png",
    "playStoreUrl": "https://play.google.com/store/apps/details?id=com.socialtool.statussaver",
    "ctaText":      "Install"
  }
]
```

---

## `ads_cadence_config`

A JSON object that overrides the cadence and timing values set in `AdConfig`. All fields are optional — omitted fields keep the `AdConfig` values.

### Schema

```json
{
  "interstitialCloseSec":         5,
  "bannerHouseCadence":           50,
  "interstitialHouseCadence":     10,
  "contentAdInterval":            4,
  "interstitialTriggerThreshold": 3
}
```

### Field reference

| Field | Type | Corresponding `AdConfig` field | Description |
|-------|------|-------------------------------|-------------|
| `interstitialCloseSec` | int | `interstitialCloseDurationSec` | Seconds before the close button appears on the house-ad full-screen interstitial. Drives the countdown ring animation. |
| `bannerHouseCadence` | int | `bannerHouseCadence` | House-ad cadence for the banner slot. |
| `interstitialHouseCadence` | int | `interstitialHouseCadence` | House-ad cadence for the interstitial slot. |
| `contentAdInterval` | int | `contentAdInterval` | House-ad injection interval for `buildAdList()`. |
| `interstitialTriggerThreshold` | int | `interstitialTriggerThreshold` | How many `onActionCompleted()` calls before an interstitial is triggered. |

### Cadence semantics

| Value | Slot behaviour |
|-------|---------------|
| `-1` | House ads disabled for this slot. FAN-only; nothing shown if FAN fails. |
| `0` | Always show house ad. FAN is never used for this slot. |
| `N > 0` | Show 1 house ad after every N FAN impressions for this slot. |

**Value priority (highest → lowest):**

1. `ads_cadence_config` RC value (applied live after fetch)
2. `AdConfig` constructor value
3. SDK built-in default

RC values are read live — cadence changes take effect on the next impression after the fetch completes, without requiring an app restart.

### Full sample

```json
{
  "interstitialCloseSec":         8,
  "bannerHouseCadence":           30,
  "interstitialHouseCadence":     5,
  "contentAdInterval":            6,
  "interstitialTriggerThreshold": 5
}
```

This config would:
- Show the close button on full-screen interstitials after 8 seconds (overriding the default 5).
- Show a house banner every 30th FAN banner impression.
- Show a house interstitial every 5th FAN interstitial.
- Inject a content ad card every 6 real items.
- Require 5 qualifying actions before an interstitial fires.

---

## RC fetch interval

The SDK fetches Remote Config with a 1-hour minimum interval. In development you can lower this:

```kotlin
val settings = remoteConfigSettings { minimumFetchIntervalInSeconds = 0 }
Firebase.remoteConfig.setConfigSettingsAsync(settings)
```

The SDK calls `fetchAndActivate()` on its own; you do not need to call it separately.
