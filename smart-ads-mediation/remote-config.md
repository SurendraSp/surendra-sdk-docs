# Smart Ads Mediation SDK — Remote Config

The mediation SDK uses the same Firebase Remote Config keys as the original Smart Ads SDK. All ad-serving behavior (cadence thresholds, house-ad visibility, in-feed frequency) is controlled from RC — no code changes or SDK releases needed.

The full RC schema and key reference is documented in the original SDK guide:

→ **[Smart Ads — Remote Config](../smart-ads/remote-config.md)**

---

## Keys specific to mediation

There are no mediation-specific RC keys. The SDK chooses between a house ad and an AdMob interstitial at runtime based on house-ad availability:

- If `house_ads` RC value is non-empty and a house ad is eligible → house ad is shown.
- Otherwise → the preloaded AdMob interstitial fires (which AdMob routes to FAN / InMobi / Google demand via the mediation waterfall).

The `ads_cadence_config` key governs how often `onActionCompleted` triggers any ad — house or network. The mediation waterfall behind network ads is configured entirely in the AdMob console, not in RC.

---

## Firebase project

Firebase project: `smart-social-tool`

Fetch current config:

```bash
firebase remoteconfig:get --project smart-social-tool -o /tmp/rc.json
```

Deploy updated config (from a directory containing `firebase.json` with `{"remoteconfig":{"template":"rc.json"}}`):

```bash
firebase deploy --only remoteconfig --project smart-social-tool
```
