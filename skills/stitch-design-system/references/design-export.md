# Design Export — Reference

Deep-dive on the export direction from `SKILL.md`: turning an existing Stitch project's `designTheme` into local files. There is no `mcp__stitch__*` export tool — this whole pipeline is `get_project` (or `list_projects` to find the project first) followed by writing files yourself. Ground truth below was verified against a live Stitch account with 13 projects.

## `designTheme` field inventory

Every key observed on a real `get_project` response. "Writable" means the field appears in the `create_design_system` / `update_design_system` write schema (see `SKILL.md` and `references/theme-enums.md`); "resolved" means it comes back populated on read but is not something you set directly — it is Stitch's own derivation from the writable fields, and the only way to get its value is to read it.

| Field | Type | Writable / resolved | Example value | Useful for on export |
|---|---|---|---|---|
| `bodyFont` | enum (font id) | Writable | `"INTER"` | Look up in `references/theme-enums.md` if you need the enum back for a round trip; for output files prefer `bodyFontFamily` below — it's already the real name. |
| `bodyFontFamily` | string (CSS family name) | Resolved | `"Inter"` | Drop straight into a `font-family` / Tailwind `fontFamily` value — no enum lookup needed. |
| `colorMode` | enum (`LIGHT` / `DARK` / …) | Writable | `"LIGHT"` | States whether the exported tokens are a light or dark set; include in DESIGN.md frontmatter or a header comment. |
| `colorVariant` | enum | Writable | `"FIDELITY"` | Documents *how* `namedColors` was derived (see `references/theme-enums.md` for the full list) — worth a line in exported prose, not a token itself. |
| `customColor` | hex string | Writable | e.g. `"#2563EB"` (a seed color; see the recipe table in `SKILL.md` for typical values) | The seed hex that `namedColors` was derived from. Export it alongside `namedColors`, don't substitute it for the resolved palette. |
| `designMd` | string (markdown) | Writable (as a notes field on a structured system) *and* the read target for export | See the excerpt in the Ground Truth section of the task that produced this file | The literal exportable `DESIGN.md` content, when present. See "The missing-`designMd` path" in `SKILL.md` — it is absent on most projects. |
| `font` | unknown | Unclear | — | **Meaning not established.** It was observed on a live project alongside `bodyFont`/`headlineFont`/`labelFont` but nothing in the tool schemas or docs explains what it selects independently of those three. Do not guess a purpose for it and do not surface it in exported artefacts as if its meaning were known — report it as raw/unexplained if you include the full JSON dump. |
| `headlineFont` | enum (font id) | Writable | `"NEWSREADER"` | Same pattern as `bodyFont` — prefer `headlineFontFamily` for output. |
| `headlineFontFamily` | string | Resolved | `"Newsreader"` | Direct CSS family name for headline text. |
| `labelFont` | enum (font id) | Writable | Not captured in the measurement behind this file; follows the same enum as `bodyFont`/`headlineFont` — see `references/theme-enums.md`. | Same role as `bodyFont`, scoped to label/UI-chrome text. |
| `labelFontFamily` | string | Resolved | Not captured in the measurement behind this file; same shape as `bodyFontFamily`. | Direct CSS family name for label text, when present. |
| `namedColors` | map (string → hex) | Resolved | 47 entries observed; sample: `{"background":"#f7f9fb","error":"#ba1a1a","error_container":"#ffdad6","inverse_on_surface":"#eff1f3","inverse_primary":"#bec6e0","inverse_surface":"#2d3133"}` | The full resolved color palette — this is the export-worthy artefact for color, not `customColor` alone. Keys are snake_case; see the normalization rule below before writing them into any output file. |
| `overrideNeutralColor` | hex string | Writable | Not captured in the measurement behind this file. | A manual override hook — if set, it means the neutral role was pinned rather than derived from `customColor`/`colorVariant`. Worth noting in exported prose if non-empty. |
| `overridePrimaryColor` | hex string | Writable | Not captured in the measurement behind this file. | Same, for the primary role. |
| `overrideSecondaryColor` | hex string | Writable | Not captured in the measurement behind this file. | Same, for the secondary role. |
| `overrideTertiaryColor` | hex string | Writable | Not captured in the measurement behind this file. | Same, for the tertiary role. |
| `roundness` | enum | Writable | `"ROUND_FOUR"` | Drives the shape/corner-radius section of exported tokens and DESIGN.md prose. See `references/theme-enums.md` for the full enum — it has no published exact px mapping (see the worked example below). |
| `spacing` | map (string → CSS length) | Resolved on read | `{"grid-margin-desktop":"48px","grid-margin-mobile":"16px","grid-margin-tablet":"24px","gutter-desktop":"24px","gutter-mobile":"12px","gutter-tablet":"16px","section-padding-lg":"80px"}` | Named spacing tokens, ready to become CSS custom properties or a Tailwind `spacing` scale. `references/theme-enums.md` shows the general shape of this map for reference; treat the concrete keys/values you read back from a real project as the authoritative export source. |
| `spacingScale` | number | Resolved on read | `2` | **Meaning not established.** Observed as a bare number; likely some kind of multiplier applied to the `spacing` map, but nothing confirms the exact semantics. Report it as-is if exporting the raw JSON; do not use it to compute or rescale other tokens. |
| `typography` | map (level name → type spec) | Resolved on read | 11 levels observed: `body-lg`, `body-md`, `body-sm`, `display-xl`, `display-xl-mobile`, `headline-lg`, `headline-lg-mobile`, `headline-md`, `headline-sm`, `label-md`, `label-sm`. See the worked example below for a full entry. | The type scale, ready to become CSS custom properties or a Tailwind `fontSize`/`lineHeight` scale. As with `spacing`, `references/theme-enums.md` documents the map's general shape; the values read from a real project are what you export. |

## Worked example

Real observed values for one project: `bodyFont: INTER` / `bodyFontFamily: "Inter"`, `headlineFont: NEWSREADER` / `headlineFontFamily: "Newsreader"`, `roundness: ROUND_FOUR`, `colorMode: LIGHT`, `colorVariant: FIDELITY`, plus the `namedColors` sample and `body-lg` typography entry from the inventory table above, plus these `spacing` entries: `grid-margin-desktop: 48px`, `grid-margin-mobile: 16px`, `grid-margin-tablet: 24px`, `gutter-desktop: 24px`, `gutter-mobile: 12px`, `gutter-tablet: 16px`, `section-padding-lg: 80px`.

`roundness` has no published px equivalent for any enum value, including `ROUND_FOUR` — if a token file needs a concrete radius, pick a value consistent with the enum's stated intent ("smallest concrete value," a sharp/precise feel — 4px is a reasonable placeholder) and flag it to the user as an assumption to confirm against the rendered Stitch UI, not as a value read from the API.

### `DESIGN.md` (synthesized — see the synthesis template below if `designMd` was absent for this project)

```markdown
---
name: Example Project
colors:
  background: '#f7f9fb'
  error: '#ba1a1a'
  error-container: '#ffdad6'
  inverse-on-surface: '#eff1f3'
  inverse-primary: '#bec6e0'
  inverse-surface: '#2d3133'
---

## Color

Light mode (`colorMode: LIGHT`), palette derived with `colorVariant: FIDELITY`
(stays close to the literal seed color across roles).

## Typography

- Headline font: Newsreader
- Body font: Inter
- Body large: 20px / 32px line-height, weight 400, -0.005em letter-spacing

## Shape

- Roundness: ROUND_FOUR — small corner radius, sharp/precise feel.
```

### CSS custom properties

```css
:root {
  /* Color — namedColors, normalized to kebab-case (see below) */
  --color-background: #f7f9fb;
  --color-error: #ba1a1a;
  --color-error-container: #ffdad6;
  --color-inverse-on-surface: #eff1f3;
  --color-inverse-primary: #bec6e0;
  --color-inverse-surface: #2d3133;

  /* Typography */
  --font-headline: "Newsreader", serif;
  --font-body: "Inter", sans-serif;
  --text-body-lg-size: 20px;
  --text-body-lg-weight: 400;
  --text-body-lg-leading: 32px;
  --text-body-lg-tracking: -0.005em;

  /* Spacing */
  --spacing-grid-margin-desktop: 48px;
  --spacing-grid-margin-tablet: 24px;
  --spacing-grid-margin-mobile: 16px;
  --spacing-gutter-desktop: 24px;
  --spacing-gutter-tablet: 16px;
  --spacing-gutter-mobile: 12px;
  --spacing-section-padding-lg: 80px;

  /* Shape — ROUND_FOUR has no published px value; 4px is a placeholder, confirm against the rendered UI */
  --radius-base: 4px;
}
```

### Tailwind theme fragment

```js
// tailwind.config.js (theme.extend), or the equivalent @theme block in Tailwind v4
export default {
  theme: {
    extend: {
      colors: {
        background: "#f7f9fb",
        error: "#ba1a1a",
        "error-container": "#ffdad6",
        "inverse-on-surface": "#eff1f3",
        "inverse-primary": "#bec6e0",
        "inverse-surface": "#2d3133",
      },
      fontFamily: {
        headline: ["Newsreader", "serif"],
        body: ["Inter", "sans-serif"],
      },
      fontSize: {
        "body-lg": ["20px", { lineHeight: "32px", letterSpacing: "-0.005em" }],
      },
      spacing: {
        "grid-margin-desktop": "48px",
        "grid-margin-tablet": "24px",
        "grid-margin-mobile": "16px",
        "gutter-desktop": "24px",
        "gutter-tablet": "16px",
        "gutter-mobile": "12px",
        "section-padding-lg": "80px",
      },
      borderRadius: {
        base: "4px", // placeholder for ROUND_FOUR — see note above
      },
    },
  },
};
```

All three artefacts above use the same six colors, the same two font families, the same `body-lg` type spec, and the same seven spacing values — keep any real export mutually consistent the same way; a token that differs between the DESIGN.md and the CSS file is a sign one of them was hand-edited after the fact and drifted.

## The `namedColors` naming problem

`namedColors` keys come back **snake_case** (`inverse_on_surface`, `error_container`). A `designMd` frontmatter block, when one exists, uses **kebab-case** (`inverse-surface`, per the Material-3-style `colors:` map in the Ground Truth example). Tailwind and CSS custom-property conventions also expect kebab-case.

**Normalization rule:** when writing `namedColors` into any output file, replace every underscore with a hyphen and otherwise leave the key alone (`inverse_on_surface` → `inverse-on-surface`, `error_container` → `error-container`). Do this once, consistently, per file.

**Do not mix the two cases in one output file.** If a project's `designMd` already exists and uses kebab-case, and you are also emitting a CSS or Tailwind token file from `namedColors`, convert `namedColors`' snake_case to kebab-case for *both* files so they agree — don't leave the raw JSON dump snake_case in one artefact and kebab-case in another describing the same colors. The one exception is a verbatim raw-JSON export (the third row in the artefact table in `SKILL.md`): that dump is explicitly a debugging reference, not a token file, so it keeps the API's native snake_case unmodified.

## Synthesis template for a missing `designMd`

When `designTheme.designMd` is empty or absent, do not fabricate prose. Build a DESIGN.md using only what the resolved tokens actually tell you:

```markdown
---
name: <project title, from the project's `title` field>
colors:
  <every namedColors entry, normalized to kebab-case>: <hex value>
---

## Color

<colorMode value> mode, palette derived with `colorVariant: <value>`
(<one-line description of that variant from references/theme-enums.md>).
<If any override*Color field is set, note which role was pinned manually.>

## Typography

- Headline font: <headlineFontFamily>
- Body font: <bodyFontFamily>
- <One line per typography level actually present, stating size / line-height / weight / letter-spacing.>

## Shape

- Roundness: <roundness enum value> — <its meaning, from references/theme-enums.md>.
```

**Fill only from resolved tokens:** the `colors:` map from `namedColors`, the mode from `colorMode`, the variant note from `colorVariant` (plus `references/theme-enums.md`'s description of it), font names from the `*FontFamily` fields, type-scale lines from `typography`, and the roundness line from `roundness` (plus its enum meaning).

**Leave out rather than invent:** a brand summary or "who this is for" sentence, component tone (flat vs. elevated, bordered vs. borderless), and a "what to avoid" section. None of these are recoverable from tokens — a real DESIGN.md's prose is authored by a person, not derivable from `designTheme`. If the user wants those sections, ask them for the content; don't guess plausible-sounding prose to fill the template.

## File-writing guidance

- Write every artefact as **UTF-8**. `designTheme.designMd` content may contain non-ASCII text — the account this reference was verified against had Russian-language screen titles elsewhere in the same project data — so if you round-trip a `designMd` string through base64 (e.g., to hand it back to `upload_design_md`), the encode/decode must preserve UTF-8 exactly. `references/design-md.md` has the exact per-OS base64 commands and the UTF-8 caveats; use those rather than re-deriving encoding steps here.
- One artefact per file: don't concatenate a DESIGN.md and a token file into a single document, even when both come from the same export pass — a user asking only for "the colors" shouldn't have to pick them out of a combined file.
