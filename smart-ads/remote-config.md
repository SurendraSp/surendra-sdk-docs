# Smart Ads SDK — Firebase Remote Config

The SDK reads two RC keys at startup and re-fetches every hour. All RC values are optional — the SDK falls back to built-in defaults if a key is missing or malformed.

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
    "titleHi":      "मेरा दूसरा ऐप",
    "subtitle":     "Short tagline shown under the title",
    "subtitleHi":   "शीर्षक के नीचे दिखाई गई छोटी टैगलाइन",
    "iconUrl":      "https://cdn.example.com/icon.png",
    "playStoreUrl": "https://play.google.com/store/apps/details?id=com.example.otherapp",
    "ctaText":      "Install Free",
    "ctaTextHi":    "मुफ्त इंस्टॉल करें"
  }
]
```

### Field reference

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `packageName` | string | yes | — | Android package name. Used to check if the app is already installed — installed apps are never shown. |
| `title` | string | yes | — | App name displayed in banner, card, and full-screen (English). |
| `titleHi` | string | no | `""` | App name in Hindi. Empty → falls back to `title`. |
| `subtitle` | string | no | `""` | Short description / tagline (English). |
| `subtitleHi` | string | no | `""` | Tagline in Hindi. Empty → falls back to `subtitle`. |
| `iconUrl` | string | no | `""` | URL of the app icon. Loaded with Coil. Empty string → placeholder tinted with `AdTheme.primary`. |
| `playStoreUrl` | string | yes | — | Full Play Store URL. Opened via `Intent.ACTION_VIEW` when the user taps the CTA or full-screen background. |
| `ctaText` | string | no | `"Install"` | Label for the CTA button (English). |
| `ctaTextHi` | string | no | `""` | CTA button label in Hindi. Empty → falls back to `ctaText`. |

> `priority` is not set in JSON — it is assigned automatically from the array index during parsing (index 0 → priority 0, highest).

### Localization

The SDK picks the correct language string automatically:

1. App-level locale set via `AppCompatDelegate.setApplicationLocales` (takes priority)
2. Device system language
3. English fallback if no Hindi translation is provided

### Priority and round-robin logic

The `HouseAdEngine` filters out already-installed apps, then:

- `nextAd()` — used for **banner and interstitial**; cycles round-robin through all eligible apps. Each call returns the next app in sequence, so impressions rotate across all uninstalled apps.
- `buildEligibleQueue()` — used for **in-feed content ads**; returns all eligible apps in priority order. The orchestrator cycles through this queue across content list injection points.

If every app in `house_ads` is already installed, no house ads are shown for any slot.

### Sample RC value

```json
[
  {
    "packageName":  "com.parihartrick.socialtool",
    "title":        "WhatsSmart - WA Status Saver",
    "titleHi":      "WhatsSmart - स्टेटस सेवर",
    "subtitle":     "Save WhatsApp status, stylish text & more",
    "subtitleHi":   "स्टेटस सेव करें, स्टाइलिश टेक्स्ट और बहुत कुछ",
    "iconUrl":      "https://play-lh.googleusercontent.com/nAmK957nWiHYe6R_7Fah7Eby5QwOLnliamufA1cRryMqlq6FlY77BhuJxl_hGvchfg=w240-h480",
    "playStoreUrl": "https://play.google.com/store/apps/details?id=com.parihartrick.socialtool",
    "ctaText":      "Install Free",
    "ctaTextHi":    "मुफ्त इंस्टॉल करें"
  },
  {
    "packageName":  "com.panchang.jyotish",
    "title":        "Jyotish - Daily Rashifal",
    "titleHi":      "ज्योतिष - दैनिक राशिफल",
    "subtitle":     "Daily horoscope & Vedic astrology",
    "subtitleHi":   "दैनिक राशिफल और वैदिक ज्योतिष",
    "iconUrl":      "https://play-lh.googleusercontent.com/ek-sLMrZsbTz2SdLXZQY--M-i1kWArPoM79BjiV5zOF7Bg1HgbdwtpE1USRRJFu3gQpX_RPh2GZZ6M1iF_wtMw=w240-h480",
    "playStoreUrl": "https://play.google.com/store/apps/details?id=com.panchang.jyotish",
    "ctaText":      "Install Free",
    "ctaTextHi":    "मुफ्त इंस्टॉल करें"
  }
]
```

---

## `ads_cadence_config`

A JSON object that overrides cadence and timing values. All fields are optional — omitted fields use the SDK built-in defaults.

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

| Field | Type | SDK default | Description |
|-------|------|-------------|-------------|
| `interstitialCloseSec` | int | `5` | Seconds before the close button appears on the house-ad full-screen interstitial. Drives the countdown ring animation. |
| `bannerHouseCadence` | int | `50` | House-ad cadence for the banner slot. |
| `interstitialHouseCadence` | int | `10` | House-ad cadence for the interstitial slot. |
| `contentAdInterval` | int | `4` | House-ad injection interval for `buildAdList()`. |
| `interstitialTriggerThreshold` | int | `3` | How many `onActionCompleted()` calls before an interstitial is triggered. |

### Cadence semantics

| Value | Slot behaviour |
|-------|---------------|
| `-1` | House ads disabled for this slot. FAN-only; nothing shown if FAN fails. |
| `0` | Always show house ad. FAN is never used for this slot. |
| `N > 0` | Show 1 house ad after every N FAN impressions for this slot. |

RC values are read live — changes take effect on the next impression after the fetch completes, without an app restart.

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
- Show the close button on full-screen interstitials after 8 seconds.
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

The SDK calls `fetch()` + `activate()` on its own; you do not need to call it separately.
