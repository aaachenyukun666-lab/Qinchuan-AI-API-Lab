# AI Relay Meter — Design Tokens

Design language: Apple / macOS. Tokens below are distilled from the Apple
analysis in the `awesome-design-md` reference (`design-md/apple/DESIGN.md`),
with **Inter** (rsms/inter) as the SF Pro substitute on non-Apple platforms and
the **ablur** multi-layer masking technique (xujunhao940/ablur) for the frosted
sticky header. See `LICENSE-THIRD-PARTY.md`.

## Color

| Token | Light | Dark | Use |
|---|---|---|---|
| `--canvas` | `#ffffff` | `#000000` | Page canvas |
| `--canvas-parchment` | `#f5f5f7` | `#161617` | Alternating surface, page background |
| `--surface` | `#ffffff` | `#1d1d1f` | Cards |
| `--surface-pearl` | `#fafafc` | `#2c2c2e` | Ghost buttons, sunken wells |
| `--tile-dark` | — | `#272729` / `#2a2a2c` | Dark surfaces |
| `--ink` | `#1d1d1f` | `#f5f5f7` | Primary text (never pure black) |
| `--ink-muted` | `#6e6e73` | `#a1a1a6` | Secondary text |
| `--ink-faint` | `#7a7a7a` | `#8e8e93` | Disabled / fine print |
| `--ink-muted-dark` | — | `#cccccc` | Muted copy on dark tiles |
| `--primary` | `#0066cc` | `#2997ff` | Action Blue; all interactive accents |
| `--primary-focus` | `#0071e3` | `#0071e3` | Focus ring |
| `--primary-on-dark` | `#2997ff` | `#2997ff` | Links on dark tiles (Sky Link Blue) |
| `--hairline` | `#e0e0e0` | `#3a3a3c` | 1px borders |
| `--divider-soft` | `rgba(0,0,0,.04)` | `rgba(255,255,255,.06)` | Soft ring "borders" |
| `--chip-translucent` | `rgba(210,210,215,.64)` | `rgba(70,70,75,.64)` | Floating circular controls |
| `--ok` | `#1d8a4e` | `#30d158` | Success |
| `--warn` | `#a86500` | `#ff9f0a` | Discrepancy verdict |
| `--danger` | `#d70015` | `#ff453a` | Failure |

**No decorative gradients.** Depth comes from surface changes and blur only —
this mirrors Apple's zero-gradient token system.

## Typography

Font stack: `"Inter var", "Inter", SF Pro Display, SF Pro Text, system-ui,
-apple-system, "Segoe UI", "PingFang SC", "Hiragino Sans GB", sans-serif`.
Inter is self-hosted (`assets/fonts/*.woff2`, weights 400/500/600/700) so the
Android WebView renders identically offline.

| Token | Size | Weight | Line height | Tracking | Use |
|---|---|---|---|---|---|
| `--type-hero` | 40px | 600 | 1.10 | −0.374px | Hero headline |
| `--type-lead` | 24px | 400 | 1.30 | −0.196px | Hero subcopy |
| `--type-tagline` | 21px | 600 | 1.19 | −0.231px | Section intro |
| `--type-body-strong` | 17px | 600 | 1.24 | −0.374px | Emphasis, stat values |
| `--type-body` | 17px | 400 | 1.47 | −0.374px | Default copy |
| `--type-caption` | 14px | 400 | 1.43 | −0.224px | Labels, buttons |
| `--type-caption-strong` | 14px | 600 | 1.29 | −0.224px | Verdict labels |
| `--type-fine` | 12px | 400 | 1.33 | −0.12px | Hints, footer |
| `--type-mono` | 12px | 400 | 1.55 | 0 | Logs, key previews |

Rules carried over from the reference:

- Weight **600** for headlines, never 700; weight **300/500 ladder absent** —
  we use 400/600 only (500 is kept solely for Inter's font loading granularity).
- Negative tracking from 17px up; **never** below 12px.
- Body reads at 17px, not 16px.
- Inter substitution: tracking nudged −0.01em at display sizes; body
  line-height tightened 1.47 → 1.44 to offset Inter's taller x-height.

## Spacing

Base unit 8px; structural steps 4 / 8 / 12 / 16 / 24 / 32 / 48; card padding
24px; section rhythm 48–80px; card gutters 20–24px. Whitespace is generous by
design — each card's headline gets at least 24px of clear air above.

## Shape

| Token | Value | Use |
|---|---|---|
| `--radius-sm` | 8px | Inputs, small buttons |
| `--radius-md` | 11px | Pearl capsules |
| `--radius-lg` | 18px | Cards |
| `--radius-pill` | 9999px | Primary CTAs, chips |

## Elevation

Exactly one shadow token, and only for genuinely floating layers:

```css
--shadow-product: 0 3px 5px rgba(0, 0, 0, 0.22), 0 0 30px rgba(0, 0, 0, 0.08);
```

Cards get **hairline borders, not shadows**. Elevation otherwise comes from the
frosted sticky header (below) and surface-color changes.

## Frosted header (ablur technique)

The sticky masthead reproduces Apple's `sub-nav-frosted`: parchment at ~80%
opacity plus `backdrop-filter: blur(20px)` saturate-boosted. Where content
scrolls under the header edge, a small stack of progressively blurred,
gradient-masked layers fades the frosted region out — the CSS slice technique
from `ablur`, transcribed to plain CSS with `mask-image` linear-gradients and a
`-webkit-mask-composite: source-over` fallback, degrading to a plain opaque
header when `backdrop-filter` is unsupported (older WebViews).

## Motion

- Micro-press: `transform: scale(0.95)` on active CTAs (the system-wide
  Apple interaction), 150ms ease-out.
- Progress bar: 250ms width ease; verdict blocks fade in 220ms.
- `prefers-reduced-motion: reduce` disables all of the above.

## Focus

Keyboard focus ring: `outline: 2px solid var(--primary-focus)` with a 2px
offset, on every interactive element — never removed.
