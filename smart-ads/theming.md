# Smart Ads SDK — Theming

The SDK's default ad composables (`HouseAdBanner`, `HouseAdCard`, `HouseAdFullScreen`) all read from an `AdTheme` that you supply in `AdConfig`. Every property is nullable — `null` falls back to the host app's ambient `MaterialTheme` automatically, so apps with a properly configured MaterialTheme can pass `AdTheme()` and get correct branding with no extra work.

---

## `AdTheme` fields

```kotlin
data class AdTheme(
    val primary:   Color? = null,   // CTA button background, icon placeholder tint
    val onPrimary: Color? = null,   // CTA button text / icon color
    val surface:   Color? = null,   // Banner / card background
    val onSurface: Color? = null,   // Title, subtitle, "Ad" badge text
    val cardShape: Shape? = null,   // Content card corner radius
    val ctaShape:  Shape? = null,   // CTA button corner radius
)
```

### Field-by-field reference

| Field | Type | Used in | MaterialTheme fallback |
|-------|------|---------|------------------------|
| `primary` | `Color?` | CTA button background; icon placeholder fill (`alpha = 0.2`) | `MaterialTheme.colorScheme.primary` |
| `onPrimary` | `Color?` | CTA button text and icon color | `MaterialTheme.colorScheme.onPrimary` |
| `surface` | `Color?` | Banner row background; content card gradient start | `MaterialTheme.colorScheme.secondaryContainer` |
| `onSurface` | `Color?` | Title text; subtitle text; "Ad" badge text (`alpha = 0.6`); "Ad" badge border (`alpha = 0.25`) | `MaterialTheme.colorScheme.onSecondaryContainer` |
| `cardShape` | `Shape?` | `HouseAdCard` outer clip shape | `RoundedCornerShape(12.dp)` |
| `ctaShape` | `Shape?` | CTA button shape in all three composables | `RoundedCornerShape(8.dp)` |

### Companion defaults

```kotlin
AdTheme.DEFAULT_CARD_SHAPE  // RoundedCornerShape(12.dp)
AdTheme.DEFAULT_CTA_SHAPE   // RoundedCornerShape(8.dp)
```

---

## How theming is propagated

`AdTheme` is distributed through the Compose composition via `CompositionLocal`:

```
AdBannerSlot / HouseAdFullScreenHost
    └── CompositionLocalProvider(LocalAdTheme provides config.theme)
            └── HouseAdBanner / HouseAdCard / HouseAdFullScreen
                    └── val theme = LocalAdTheme.current
```

Every nested SDK composable reads `LocalAdTheme.current` and resolves `null` fields against `MaterialTheme` at composition time. This means:
- You only need to set `AdTheme` once in `AdConfig`.
- Theme updates in `AdConfig` do not hot-reload at runtime (the theme is fixed at `init()`).
- If you use the slot API (`houseAdContent`, `content` lambdas), your composable runs inside the same `CompositionLocalProvider`, so `LocalAdTheme.current` is available to you too.

---

## Examples

### Minimal — rely on MaterialTheme

If your app has a MaterialTheme with `primary`, `onPrimary`, `secondaryContainer`, and `onSecondaryContainer` set, no custom theme is needed:

```kotlin
AdConfig(
    ...
    theme = AdTheme()   // all nulls → MaterialTheme fallbacks
)
```

### Brand colors only

```kotlin
AdConfig(
    ...
    theme = AdTheme(
        primary   = Color(0xFF1B8C3E),
        onPrimary = Color.White,
    )
)
```

### Full brand override

```kotlin
AdConfig(
    ...
    theme = AdTheme(
        primary   = Color(0xFF1B8C3E),   // green CTA background
        onPrimary = Color.White,          // white CTA text
        surface   = Color(0xFFE8F5E9),   // light green banner/card background
        onSurface = Color(0xFF1B5E20),   // dark green text
        cardShape = RoundedCornerShape(16.dp),   // rounded content cards
        ctaShape  = RoundedCornerShape(50.dp),   // pill-shaped CTA button
    )
)
```

### Dark-mode app

```kotlin
AdConfig(
    ...
    theme = AdTheme(
        primary   = Color(0xFF4CAF50),
        onPrimary = Color.Black,
        surface   = Color(0xFF1E2D1E),
        onSurface = Color(0xFFB2DFBA),
        ctaShape  = RoundedCornerShape(12.dp),
    )
)
```

---

## Color resolution per composable

### `HouseAdBanner` (50 dp tall)

| Colour resolved | Source (if null → fallback) |
|-----------------|-----------------------------|
| CTA button background | `primary` → `MaterialTheme.colorScheme.primary` |
| CTA button text | `onPrimary` → `MaterialTheme.colorScheme.onPrimary` |
| Row background | `surface` → `MaterialTheme.colorScheme.secondaryContainer` |
| Title / subtitle text | `onSurface` → `MaterialTheme.colorScheme.onSecondaryContainer` |
| "Ad" badge | `onSurface` at alpha 0.6 / border at alpha 0.25 |
| CTA corner | `ctaShape` → `RoundedCornerShape(8.dp)` |

### `HouseAdCard` (square, 1:1 aspect ratio)

| Colour resolved | Source (if null → fallback) |
|-----------------|-----------------------------|
| Card gradient start | `surface` → `MaterialTheme.colorScheme.primaryContainer` |
| Card gradient end | `surface` → `MaterialTheme.colorScheme.secondaryContainer` |
| Card clip shape | `cardShape` → `RoundedCornerShape(12.dp)` |
| CTA button background | `primary` → `MaterialTheme.colorScheme.primary` |
| CTA button text | `onPrimary` → `MaterialTheme.colorScheme.onPrimary` |
| Title / subtitle text | `onSurface` → `MaterialTheme.colorScheme.onPrimaryContainer` |
| "Ad" badge | `onSurface` at alpha 0.5 / border at alpha 0.2 |
| CTA corner | `ctaShape` → `RoundedCornerShape(8.dp)` |

### `HouseAdFullScreen` (full-screen overlay)

| Element | Color |
|---------|-------|
| Scrim background | `Color.Black.copy(alpha = 0.93f)` — not themeable |
| Subtitle header ("From the same developer") | `Color.White.copy(alpha = 0.6f)` — not themeable |
| Title / app name | `Color.White` — not themeable |
| "Ad" badge | `Color.White` at alpha 0.55 / border at alpha 0.25 — not themeable |
| CTA button background | `primary` → `MaterialTheme.colorScheme.primary` |
| CTA button text | `onPrimary` → `MaterialTheme.colorScheme.onPrimary` |
| CTA corner | `ctaShape` → `RoundedCornerShape(14.dp)` *(note: 14 dp, not the default 8 dp)* |
| Countdown ring | `primary` → `MaterialTheme.colorScheme.primary` |
| Close button background | `Color.White.copy(alpha = 0.15f)` — not themeable |
| Close button icon | `Color.White` — not themeable |

> Full-screen text is always white because the scrim is always dark. These values are not controlled by `AdTheme`.

---

## Slot API for full control

If `AdTheme` isn't enough, use the slot API to replace the SDK default UI entirely:

```kotlin
// Custom banner
AdBannerSlot(
    houseAdContent = { config ->
        MyBanner(config)   // your composable
    }
)

// Custom interstitial
HouseAdFullScreenHost(
    content = { config, onDismiss ->
        MyInterstitial(config, onDismiss)   // you must call onDismiss
    }
)

// Custom content card
is AdListItem.HouseAd -> MyAdCard(item.config)
```

When using the slot API, `LocalAdTheme.current` is still available inside your lambda if you want to read the theme values.
