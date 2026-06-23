# Surendra SDK Docs

Public documentation for all Android SDKs published under [SurendraSp](https://github.com/SurendraSp).

---

## SDKs

| SDK | Version | Description |
|-----|---------|-------------|
| [Smart Ads](smart-ads/getting-started.md) | `1.0.6-RC3` | FAN + house-ad cross-sell banner, interstitial, and in-feed ads with progressive cadence ramping and Firebase Remote Config |
| [Smart Ads Mediation](smart-ads-mediation/getting-started.md) | `1.0.0-RC1` | AdMob mediation variant — banner and interstitial via AdMob waterfall (FAN, InMobi, Google demand) with house-ad cross-sell preserved |

---

## Quick links

### Smart Ads SDK

| Doc | Description |
|-----|-------------|
| [Getting Started](smart-ads/getting-started.md) | Installation, credentials, `AdConfig` setup |
| [API Reference](smart-ads/api-reference.md) | All public classes, methods, composables, and flows |
| [Remote Config](smart-ads/remote-config.md) | RC keys, JSON schemas, cadence semantics |
| [Theming](smart-ads/theming.md) | `AdTheme` fields, MaterialTheme fallback table |
| [Changelog](smart-ads/changelog.md) | Version history |
| [Agent Integration](smart-ads/agent-integration.md) | Self-contained context file for AI-assisted integration |

### Smart Ads Mediation SDK

| Doc | Description |
|-----|-------------|
| [Getting Started](smart-ads-mediation/getting-started.md) | Installation, AdMob App ID, `MediationConfig` setup |
| [API Reference](smart-ads-mediation/api-reference.md) | `AdsSDK`, `MediationConfig`, composables, analytics events |
| [Mediation Setup](smart-ads-mediation/mediation-setup.md) | AdMob dashboard — adding FAN and InMobi mediation networks |
| [Remote Config](smart-ads-mediation/remote-config.md) | RC keys (shared with Smart Ads SDK) |
| [Changelog](smart-ads-mediation/changelog.md) | Version history, migration guide from `ads-sdk` |

---

## Maven coordinates

### Smart Ads SDK (FAN-direct)

```
com.github.SurendraSp:smart-ads:1.0.6-RC3
```

### Smart Ads Mediation SDK (AdMob mediation)

```
io.surendrasp:ads-sdk-mediation:1.0.0-RC1
```

Registry: `https://maven.pkg.github.com/SurendraSp/SmartAdsSDK`
