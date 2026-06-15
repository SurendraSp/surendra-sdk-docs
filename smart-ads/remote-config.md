# Smart Ads SDK — Firebase Remote Config

The SDK reads two RC keys at startup and re-fetches every hour.

---

## RC keys

| Key | RC Type | Description |
|-----|---------|-------------|
| `house_ads` | JSON string (array) | List of cross-sell apps to advertise |
| `ads_cadence_config` | JSON string (object) | Cadence and timing overrides |

## FAN-only fallback (1.0.6-RC and later)

Both RC keys are **opt-in**. House ads are not shown unless both are configured:

- `house_ads` missing or empty → no house-ad inventory → every slot routes to FAN.
- `ads_cadence_config` missing → all house-ad cadences default to `-1` (disabled), so banner/interstitial/content slots only consider FAN.

The SDK ships with **no** built-in house ads and **no** built-in house-ad cadence. Apps that only want FAN can install the SDK and ignore Remote Config entirely.

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

A JSON object that overrides cadence and timing values. All fields are optional. Two flavours of fields exist:

- **Flat fields** (banner cadence, interstitial cadence, close timer) — take effect immediately at their configured value.
- **Ramped fields** (`interstitialTriggerThreshold`, `contentAdInterval`) — linearly interpolate from a `*Start` value down to the target value as the user accumulates sessions and actions. Brand-new users see fewer ads; engaged users converge on the target rate. *Omit the `*Start` key to disable the ramp — start defaults to target → behaviour identical to pre-1.0.6.*

### Schema

```json
{
  "interstitialCloseSec":              5,
  "bannerHouseCadence":                50,
  "interstitialHouseCadence":          10,

  "contentAdInterval":                 5,
  "contentAdIntervalStart":            10,

  "interstitialTriggerThreshold":      3,
  "interstitialTriggerThresholdStart": 20,

  "loyalSessionCount":                 15,
  "loyalActionCount":                  100,
  "sessionDebounceMinutes":            20
}
```

### Field reference

| Field | Type | SDK default | Description |
|-------|------|-------------|-------------|
| `interstitialCloseSec` | int | `5` | Seconds before the close button appears on the house-ad full-screen interstitial. Drives the countdown ring animation. |
| `bannerHouseCadence` | int | `-1` | House-ad cadence for the banner slot. **Default disables house ads** — set explicitly to enable. |
| `interstitialHouseCadence` | int | `-1` | House-ad cadence for the interstitial slot. **Default disables house ads** — set explicitly to enable. |
| `contentAdInterval` | int | `-1` | Target house-ad injection interval for `buildAdList()`. **Default disables content ads.** |
| `contentAdIntervalStart` | int | = `contentAdInterval` | Starting interval for new users (ramped down to target). Use a larger number than the target to ease new users in. |
| `interstitialTriggerThreshold` | int | `3` | Target action count between interstitial firings. |
| `interstitialTriggerThresholdStart` | int | = `interstitialTriggerThreshold` | Starting threshold for new users (ramped down to target). Use a larger number to start gentler. |
| `loyalSessionCount` | int | `15` | Cold-start sessions required to reach "loyal" — at which the ramp completes and target cadence applies. |
| `loyalActionCount` | int | `100` | Total actions required to reach "loyal". Whichever signal hits first graduates the user. |
| `sessionDebounceMinutes` | int | `20` | A cold start only counts as a new session if more than this many minutes have passed since last activity. Prevents background ↔ foreground bounces from inflating the session count. |

### Cadence semantics

| Field | Value | Behaviour |
|-------|-------|-----------|
| `bannerHouseCadence` / `interstitialHouseCadence` | `-1` | House ads disabled. FAN-only; nothing shown if FAN fails. |
| `bannerHouseCadence` / `interstitialHouseCadence` | `0` | Always show house ad. FAN is never used for this slot. |
| `bannerHouseCadence` / `interstitialHouseCadence` | `N > 0` | Show 1 house ad after every N FAN impressions. |
| `contentAdInterval` | `-1` or `0` | No content ads injected. List returned as-is. |
| `contentAdInterval` | `N > 0` | Inject 1 house-ad card after every N real content items. |

### How the ramp works

For ramped fields, the SDK computes user engagement progress on every decision:

```
progress = min(1.0, max(sessionCount / loyalSessionCount, totalActions / loyalActionCount))
```

Whichever signal advances faster wins — a power user racks up 100 actions in a weekend without waiting for 15 cold starts, while a daily-but-shallow user gets there via session count. The effective cadence is then:

```
effective = start + (target - start) * progress
```

…rounded **up** for `interstitialTriggerThreshold` (favours the user — fewer ads during the ramp) and **nearest** for `contentAdInterval`. Result is clamped to `≥ 1`. The math short-circuits to `target` whenever `start == target`, so the no-ramp case has zero overhead.

### Force-loyal override (QA)

`AdsSDK.setForceLoyalTier(true)` pins `progress = 1.0` so QA can verify target-cadence behaviour without waiting for a real user to graduate. In-memory only; resets on process restart.

### Full sample

```json
{
  "interstitialCloseSec":              8,
  "bannerHouseCadence":                30,
  "interstitialHouseCadence":          5,

  "contentAdInterval":                 5,
  "contentAdIntervalStart":            10,

  "interstitialTriggerThreshold":      3,
  "interstitialTriggerThresholdStart": 20,

  "loyalSessionCount":                 15,
  "loyalActionCount":                  100,
  "sessionDebounceMinutes":            20
}
```

This config would:
- Show the close button on full-screen interstitials after 8 seconds.
- Show a house banner every 30th FAN banner impression.
- Show a house interstitial every 5th FAN interstitial.
- **New user**: inject 1 content ad per 10 items; fire an interstitial every 20 actions.
- **Loyal user** (≥15 sessions or ≥100 actions): inject 1 content ad per 5 items; fire an interstitial every 3 actions.
- Sessions are debounced — a background-then-foreground within 20 minutes does not count as a new session.

RC values are read live — changes take effect on the next impression after the fetch completes, without an app restart.

---

## RC fetch interval

The SDK fetches Remote Config with a 1-hour minimum interval. In development you can lower this:

```kotlin
val settings = remoteConfigSettings { minimumFetchIntervalInSeconds = 0 }
Firebase.remoteConfig.setConfigSettingsAsync(settings)
```

The SDK calls `fetch()` + `activate()` on its own; you do not need to call it separately.
