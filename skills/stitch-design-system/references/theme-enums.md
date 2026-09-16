# Stitch Theme Enums — Reference

Lookup tables for every closed enum used in `DesignTheme` and related Stitch MCP tools. Consult this file before emitting any value not already copied verbatim from a recipe in `SKILL.md`. Do not use a value that is not listed here.

## Fonts (`headlineFont`, `bodyFont`, `labelFont`)

All 68 values, in API order. "Notes" flags the three deprecated values and faces with obvious special-purpose use (display-only, monospace, condensed).

| Enum value | Family name | Classification | Notes |
|---|---|---|---|
| `FONT_UNSPECIFIED` | — | — | Sentinel; do not set explicitly, means "let Stitch choose" |
| `BE_VIETNAM_PRO` | Be Vietnam Pro | sans | Geometric humanist sans; headline or body |
| `EPILOGUE` | Epilogue | sans | Grotesque sans; headline or body |
| `INTER` | Inter | sans | UI workhorse; excellent body font |
| `LEXEND` | Lexend | sans | Designed to improve reading proficiency; strong body/accessible choice |
| `MANROPE` | Manrope | sans | Semi-condensed geometric sans; headline or body |
| `NEWSREADER` | Newsreader | serif | Literary text serif; warm headline face |
| `NOTO_SERIF` | Noto Serif | serif | Broad-coverage transitional serif; safe body serif |
| `PLUS_JAKARTA_SANS` | Plus Jakarta Sans | sans | Friendly geometric sans; corporate-clean headline |
| `PUBLIC_SANS` | Public Sans | sans | Neutral civic-grade sans; body-friendly |
| `SPACE_GROTESK` | Space Grotesk | sans | Quirky proportions; distinctive headline |
| `SPLINE_SANS` | Spline Sans | sans | Rounded-terminal grotesque; headline or UI |
| `WORK_SANS` | Work Sans | sans | Warm optically-sized grotesque; versatile body |
| `DOMINE` | Domine | serif | Sturdy serif; heading weight |
| `LIBRE_CASLON_TEXT` | Libre Caslon Text | serif | Classic Caslon-style text serif |
| `EB_GARAMOND` | EB Garamond | serif | Old-style Garamond; elegant/literary |
| `LITERATA` | Literata | serif | Literary serif built for reading; good body serif |
| `SOURCE_SERIF_4` | Source Serif 4 | serif | Adobe text serif; pairs with Source Sans 3 |
| `SOURCE_SERIF_FOUR` | Source Serif Four | serif | **Deprecated** — use `SOURCE_SERIF_4` |
| `MONTSERRAT` | Montserrat | sans | Geometric sans; popular display/headline face |
| `METROPHOBIC` | Metrophobic | sans | Light geometric sans; headline/display use |
| `METROPOLIS` | Metropolis | sans | **Deprecated** — no direct successor listed; prefer `MONTSERRAT` or `PLUS_JAKARTA_SANS` |
| `SOURCE_SANS_3` | Source Sans 3 | sans | Adobe's neutral UI sans; body-friendly |
| `SOURCE_SANS_THREE` | Source Sans Three | sans | **Deprecated** — use `SOURCE_SANS_3` |
| `NUNITO_SANS` | Nunito Sans | sans | Rounded-terminal humanist sans; friendly body |
| `ARIMO` | Arimo | sans | Metric-compatible neutral sans; neutral body |
| `HANKEN_GROTESK` | Hanken Grotesk | sans | Grotesque sans; headline or body |
| `RUBIK` | Rubik | sans | Rounded-corner grotesque; friendly headline/body |
| `GEIST` | Geist | sans | Crisp modern UI sans; SaaS/dashboard default |
| `DM_SANS` | DM Sans | sans | Low-contrast geometric sans; clean UI body |
| `IBM_PLEX_SANS` | IBM Plex Sans | sans | Technical/engineering-flavored sans |
| `SORA` | Sora | sans | Rounded geometric sans; modern headline |
| `ANYBODY` | Anybody | display sans | Expressive variable display face; **avoid for body** |
| `ANTON` | Anton | display | Ultra-bold condensed display; headline only, **never body** |
| `ARCHIVO_NARROW` | Archivo Narrow | condensed sans | Space-efficient condensed sans; headline or tight labels |
| `ATKINSON_HYPERLEGIBLE_NEXT` | Atkinson Hyperlegible Next | sans | Designed for maximum legibility/low vision; best accessible body font |
| `BARLOW_CONDENSED` | Barlow Condensed | condensed sans | Condensed grotesque; headline/labels |
| `BEBAS_NEUE` | Bebas Neue | display | All-caps condensed display; headline only, **never body** |
| `BODONI_MODA` | Bodoni Moda | display serif | High-contrast didone; luxury/fashion headline, avoid small body sizes |
| `BRICOLAGE_GROTESQUE` | Bricolage Grotesque | display sans | Expressive variable grotesque; headline |
| `CHIVO` | Chivo | sans | Grotesque with condensed variants; headline or body |
| `CLIMATE_CRISIS` | Climate Crisis | display | Novelty/variable display face; headline-only, **unsuitable for body** |
| `COMFORTAA` | Comfortaa | display sans | Rounded geometric; soft/playful headline |
| `COURIER_PRIME` | Courier Prime | monospace | Typewriter monospace; code/screenplay contexts |
| `FIRA_SANS` | Fira Sans | sans | Humanist sans; body-friendly |
| `GOOGLE_SANS` | Google Sans | sans | Google product UI sans; clean headline/body |
| `GOOGLE_SANS_CODE` | Google Sans Code | monospace | Code-oriented monospace |
| `GOOGLE_SANS_FLEX` | Google Sans Flex | sans | Variable optical-size Google Sans; flexible headline/body |
| `GOOGLE_SANS_MONO` | Google Sans Mono | monospace | Monospace for code/technical UI |
| `GOOGLE_SANS_TEXT` | Google Sans Text | sans | Text-optimized Google Sans; body use |
| `IBM_PLEX_SERIF` | IBM Plex Serif | serif | Technical serif companion to Plex Sans |
| `JETBRAINS_MONO` | JetBrains Mono | monospace | Developer-tool monospace; code contexts |
| `KARLA` | Karla | sans | Grotesque with rounded details; body-friendly |
| `LIBRE_FRANKLIN` | Libre Franklin | sans | American gothic sans; headline or body |
| `MERRIWEATHER` | Merriweather | serif | Sturdy screen-legible serif; body or headline |
| `NOTO_SANS` | Noto Sans | sans | Broad-coverage neutral sans; safe default body |
| `OPEN_SANS` | Open Sans | sans | Ubiquitous neutral humanist sans; safe body |
| `OSWALD` | Oswald | condensed sans | Condensed display gothic; headline only |
| `OUTFIT` | Outfit | sans | Geometric sans; clean modern headline/UI |
| `PLAYFAIR_DISPLAY` | Playfair Display | display serif | High-contrast didone; editorial/luxury headline, **not for body** |
| `POIRET_ONE` | Poiret One | display | Ultra-thin art-deco display; headline only, low legibility at small sizes |
| `QUESTRIAL` | Questrial | sans | Single-weight geometric sans; headline/UI |
| `QUICKSAND` | Quicksand | display sans | Rounded geometric; playful headline |
| `RALEWAY` | Raleway | sans | Elegant thin-weighted geometric sans; headline-leaning |
| `ROBOTO_FLEX` | Roboto Flex | sans | Variable-axis workhorse sans; flexible headline/body |
| `SPACE_MONO` | Space Mono | monospace | Quirky monospace; code/display accents |
| `SYNE` | Syne | display sans | Bold experimental grotesque; headline only |
| `VOLLKORN` | Vollkorn | serif | Warm book-text serif; body-friendly |

Deprecated values (do not emit): `SOURCE_SERIF_FOUR`, `METROPOLIS`, `SOURCE_SANS_THREE`.

Display-only / avoid for body text: `ANTON`, `BEBAS_NEUE`, `CLIMATE_CRISIS`, `POIRET_ONE`, `OSWALD`, `PLAYFAIR_DISPLAY`, `BODONI_MODA`, `ANYBODY`, `SYNE`, `COMFORTAA`, `QUICKSAND`, `BRICOLAGE_GROTESQUE` (all are legible at headline sizes but lose clarity or feel decorative set small).

Monospace (use for code/technical UI, not prose): `JETBRAINS_MONO`, `SPACE_MONO`, `GOOGLE_SANS_MONO`, `GOOGLE_SANS_CODE`, `COURIER_PRIME`.

Condensed (space-efficient, headline/label use): `ARCHIVO_NARROW`, `BARLOW_CONDENSED`, `OSWALD`.

## `colorVariant`

| Enum value | Palette character |
|---|---|
| `COLOR_VARIANT_UNSPECIFIED` | Unset; Stitch chooses a default derivation |
| `MONOCHROME` | Single-hue, grayscale-leaning palette derived from the seed color |
| `NEUTRAL` | Low-chroma, desaturated palette; subtle, professional color presence |
| `TONAL_SPOT` | Soft, muted accent derived from the seed; general-purpose safe default |
| `VIBRANT` | High-chroma, saturated palette; energetic |
| `EXPRESSIVE` | Wide hue spread across UI roles; lively and varied |
| `FIDELITY` | Stays closest to the literal seed color across roles; high color fidelity |
| `CONTENT` | Derives the palette to complement provided content/imagery colors |
| `RAINBOW` | Broad multi-hue spread across UI roles |
| `FRUIT_SALAD` | Playful multi-color palette; more saturated and higher role-to-role contrast than `RAINBOW` |

## `roundness`

`roundness` selects a **four-step border-radius scale** (`DEFAULT`, `lg`, `xl`, `full`), not a single pixel value. The enum name approximates the scale's `lg` step — it is not the radius applied everywhere. The most reliable source for the actual values is a generated screen's own inline `<script id="tailwind-config">` — read `theme.extend.borderRadius` from it directly. The table below is the fallback, measured 2026-09-16 from `theme.extend.borderRadius` on one screen from each of three real Stitch projects:

| Enum value | Meaning | `DEFAULT` | `lg` | `xl` | `full` |
|---|---|---|---|---|---|
| `ROUNDNESS_UNSPECIFIED` | Unset; Stitch chooses a default | — | — | — | — |
| `ROUND_TWO` | **Deprecated, unused** — do not use | — | — | — | — |
| `ROUND_FOUR` | Small corner radius; sharp/precise feel. Smallest concrete value available | 0.125rem (2px) | 0.25rem (4px) | 0.5rem (8px) | 0.75rem (12px) |
| `ROUND_EIGHT` | Medium corner radius; standard modern UI default | 0.25rem (4px) | 0.5rem (8px) | 0.75rem (12px) | 9999px |
| `ROUND_TWELVE` | Large corner radius ("Round 12 or full" per tool docs) | 0.5rem (8px) | 1rem (16px) | 1.5rem (24px) | 9999px |
| `ROUND_FULL` | Fully rounded / pill-shaped corners | not observed in the measured sample — no sample project used it; read the screen's own `tailwind-config` rather than guessing | | | |

Note that `rounded-full` is not a guaranteed pill under every `roundness` value: under `ROUND_FOUR` it measures 12px, not `9999px`. Only `ROUND_EIGHT` and `ROUND_TWELVE` make `full` a true pill in the measured sample.

## `deviceType` (used by `create_design_system_from_design_md`)

| Enum value | Meaning |
|---|---|
| `DEVICE_TYPE_UNSPECIFIED` | Unset; Stitch chooses |
| `MOBILE` | Mobile device frame/layout |
| `DESKTOP` | Desktop layout |
| `TABLET` | Tablet layout |
| `AGNOSTIC` | Device-agnostic layout |

## `modelId`

An **optional** parameter on `generate_screen_from_text`, `edit_screens`, and `generate_variants`. It selects the generation model; it is not part of `DesignTheme` and is not accepted by any design-system tool.

| Enum value | Meaning |
|---|---|
| `MODEL_ID_UNSPECIFIED` | Unset; Stitch chooses the default model. Prefer this (or omit the field entirely) unless the user asks for a specific model. |
| `GEMINI_3_8_FLASH` | Higher-capability generation model. |
| `GEMINI_3_5_FLASH_LITE` | Lighter, faster model. |

The Stitch documentation publishes no quality benchmark distinguishing the two named models for screen generation. Omit `modelId` unless the user explicitly names a model or asks to trade quality against speed. Never emit a value outside this list.

## `spacing`

A map of spacing token name → string value. Both keys and values are strings; values are CSS-length strings.

```json
"spacing": {
  "xs": "4px",
  "sm": "8px",
  "md": "16px",
  "lg": "24px",
  "xl": "32px"
}
```

## `typography`

A map of typography level name → object of optional CSS-value strings: `fontFamily`, `fontSize`, `fontWeight`, `letterSpacing`, `lineHeight`.

```json
"typography": {
  "display-lg": {
    "fontFamily": "PLUS_JAKARTA_SANS",
    "fontSize": "48px",
    "fontWeight": "700",
    "letterSpacing": "-0.02em",
    "lineHeight": "1.1"
  },
  "body-md": {
    "fontFamily": "INTER",
    "fontSize": "16px",
    "fontWeight": "400",
    "letterSpacing": "0em",
    "lineHeight": "1.5"
  }
}
```
