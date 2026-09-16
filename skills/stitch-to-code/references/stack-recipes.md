# Stack Recipes — Reference

Concrete, copy-ready recipes for translating a Stitch design system's theme tokens and a Stitch screen's content into code, organized by target stack. Read `SKILL.md` first to determine which recipe applies — this file assumes Step 1 (stack detection) is already done.

Every recipe below works from the same worked example so the recipes are directly comparable:

```
customColor:  "#4f46e5"
headlineFont: SPACE_GROTESK
bodyFont:     INTER
roundness:    ROUND_TWELVE
colorMode:    LIGHT
```

Framework APIs cited here (Tailwind v4 `@theme`, `next/font/google`, shadcn/ui's CSS-variable theme) reflect their documented behavior as of early-to-mid 2026. Tailwind and shadcn/ui both ship frequent minor releases — if the installed version in the target project behaves differently from what's described here, trust the installed version's own docs/generated files over this reference, and say so.

---

## Tailwind CSS v4 (`@theme` in CSS)

Tailwind v4 configures itself from CSS, not a JS config file. There is no `tailwind.config.ts` to edit (unless the project explicitly kept one for compatibility) — the theme lives in a `@theme` block inside the CSS file that already contains `@import "tailwindcss";` (commonly `app/globals.css` or `src/index.css`).

Any `--color-*`, `--font-*`, `--radius-*`, or `--spacing-*` variable declared inside `@theme` automatically generates matching utility classes — declare `--color-primary` and `bg-primary` / `text-primary` / `border-primary` become available with no further config.

```css
@import "tailwindcss";

@theme {
  /* colors — from customColor + overrides */
  --color-primary: #4f46e5;
  --color-primary-foreground: #ffffff;
  --color-secondary: #eef2ff;
  --color-secondary-foreground: #312e81;

  /* fonts — from headlineFont / bodyFont */
  --font-headline: "Space Grotesk", sans-serif;
  --font-body: "Inter", sans-serif;

  /* radius — from roundness: ROUND_TWELVE. This is the measured ROUND_TWELVE scale
     (see "roundness → radius scale" near the end of this file); prefer reading
     theme.extend.borderRadius from the screen's own tailwind-config when available. */
  --radius: 8px;       /* DEFAULT */
  --radius-lg: 16px;
  --radius-xl: 24px;
  --radius-full: 9999px;
}
```

Use it in markup as ordinary Tailwind utilities: `className="bg-primary text-primary-foreground rounded font-headline"` (`rounded` maps to the unsuffixed `--radius` / `DEFAULT` step; use `rounded-lg`, `rounded-xl`, or `rounded-full` for the other three steps).

**Dark mode (`colorMode: DARK` on a project that wants both modes):** Tailwind v4 dropped the old `darkMode: 'class'` config key. Define a custom variant once, then override the same `--color-*` variables under it:

```css
@custom-variant dark (&:where(.dark, .dark *));

@theme {
  --color-primary: #4f46e5;
}

.dark {
  --color-primary: #818cf8;
}
```

If the target project already has its own dark-mode variant or a different mechanism (a theme provider, `prefers-color-scheme`), reuse it — don't introduce a second competing one. Verify the exact `@custom-variant` syntax against the installed Tailwind version if the project's own CSS doesn't already show it, since this is one of the newer v4-specific APIs and worth double-checking rather than assuming.

**Font loading:** add a `<link>` or `@import` for the Google Fonts faces referenced by `--font-headline` / `--font-body` (see the font table below) above the `@theme` block, or load them via `next/font/google` if the project is Next.js (see that recipe).

---

## Tailwind CSS v3 (`tailwind.config.ts`)

```ts
// tailwind.config.ts
import type { Config } from "tailwindcss";

export default {
  darkMode: "class", // colorMode: LIGHT project that also supports DARK via a class toggle
  content: ["./app/**/*.{ts,tsx}", "./components/**/*.{ts,tsx}"],
  theme: {
    extend: {
      colors: {
        primary: {
          DEFAULT: "#4f46e5",
          foreground: "#ffffff",
        },
        secondary: {
          DEFAULT: "#eef2ff",
          foreground: "#312e81",
        },
      },
      fontFamily: {
        headline: ["Space Grotesk", "sans-serif"],
        body: ["Inter", "sans-serif"],
      },
      borderRadius: {
        // roundness: ROUND_TWELVE — this is the measured ROUND_TWELVE scale (see
        // "roundness → radius scale" near the end of this file); prefer reading
        // theme.extend.borderRadius from the screen's own tailwind-config when available.
        DEFAULT: "8px",
        lg: "16px",
        xl: "24px",
        full: "9999px",
      },
    },
  },
} satisfies Config;
```

Usage is identical at the class-name level to v3's always-been-standard form: `className="bg-primary text-primary-foreground rounded font-headline"` (`rounded-lg`, `rounded-xl`, `rounded-full` for the other three steps).

For `colorMode: DARK`, either flip `darkMode` handling at the app root (add/remove a `dark` class on `<html>`) or, if the project already manages this, don't touch the mechanism — just make sure the new `primary`/`secondary` colors have sensible dark-mode companions defined via `dark:` variants or CSS variables referenced from the Tailwind config's `colors`.

---

## React + Tailwind: component file shape

Match whatever export/props style Step 1 found in the project's existing components. Absent an existing convention to match (e.g. a brand-new project), this is a reasonable default:

```tsx
// components/PricingCard.tsx
import { cn } from "@/lib/utils"; // or "clsx"/"classnames" directly if that's what the project uses

interface PricingCardProps {
  title: string;
  price: string;
  featured?: boolean;
  className?: string;
}

export function PricingCard({ title, price, featured, className }: PricingCardProps) {
  return (
    <div
      className={cn(
        "rounded-md border p-6 font-body",
        featured && "border-primary bg-primary/5",
        className
      )}
    >
      <h3 className="font-headline text-lg">{title}</h3>
      <p className="text-2xl font-semibold">{price}</p>
    </div>
  );
}
```

Notes:
- `cn` here is the conventional shadcn/ui helper (`clsx` + `tailwind-merge`), imported from `@/lib/utils`. Only use it if the project actually has it — confirm the import path from Step 1, don't assume `@/lib/utils` exists.
- Conditional classes (`featured && "..."`) go through `cn`, not string concatenation, so conflicting Tailwind classes resolve correctly when `tailwind-merge` is present.
- If the project uses named vs. default exports differently than shown, match the project, not this example.

---

## Next.js App Router + shadcn/ui

shadcn/ui themes through CSS custom properties, not Tailwind config colors directly. In a shadcn project (detected via `components.json`), the color tokens live in the root CSS file as `:root` / `.dark` variable pairs, then get exposed to Tailwind as color utilities — in Tailwind v4 shadcn projects, via an `@theme inline` block that aliases each `--color-*` Tailwind variable to the corresponding `--background`/`--primary`/etc. variable; in Tailwind v3 shadcn projects, via `hsl(var(--primary))`-style color entries in `tailwind.config.ts`. Check which form the project's own generated CSS already uses (from `npx shadcn@latest init`) rather than assuming — the two forms are not interchangeable, and shadcn's exact variable set has evolved across versions, so treat the list below as the common core and verify against what's actually in the project's CSS.

**Tailwind v3-era shadcn** — HSL channel triplets, consumed as `hsl(var(--primary))` via `tailwind.config.ts`:

```css
:root {
  --background: 0 0% 100%;
  --foreground: 240 10% 10%;
  --primary: 243 75% 59%;        /* from customColor #4f46e5 */
  --primary-foreground: 0 0% 100%;
  --secondary: 234 89% 96%;
  --secondary-foreground: 243 47% 20%;
  --border: 240 6% 90%;
  --input: 240 6% 90%;
  --ring: 243 75% 59%;
  --radius: 1rem;                /* roundness: ROUND_TWELVE → lg is 1rem (16px), per the measured scale below; not shadcn's current default of 0.625rem — this value is derived from the Stitch roundness token, so it is correct here, not a mistake */
}

.dark {
  --background: 240 10% 8%;
  --foreground: 0 0% 98%;
  --primary: 243 75% 68%;
  --primary-foreground: 240 10% 10%;
  /* ...border/input/ring dark equivalents */
}
```

**Tailwind v4-era shadcn** — full color values, consumed directly via `@theme inline`:

```css
:root {
  --background: oklch(1 0 0);
  --foreground: oklch(0.15 0 0);
  --primary: oklch(0.51 0.23 277);   /* from customColor #4f46e5 */
  --primary-foreground: oklch(1 0 0);
  --radius: 1rem;                    /* roundness: ROUND_TWELVE → lg is 1rem (16px), per the measured scale below */
}

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
}
```

**Do not mix the two.** Read the project's existing root CSS and match whichever form is already there — wrapping an `oklch()` value in `hsl()`, or using a bare HSL triplet directly as a color value, silently produces wrong colors rather than an error.

shadcn/ui components consume `--radius` through derived values (`--radius-sm`, `--radius-md`, `--radius-lg` computed as offsets from `--radius` in newer shadcn CSS, or referenced directly in older setups) — don't hand-edit every component's radius; set `--radius` once and let the primitives inherit it, matching whatever derivation the project's own CSS already defines.

**Loading the two Google Fonts with `next/font/google`:**

```ts
// app/layout.tsx
import { Inter, Space_Grotesk } from "next/font/google";

const bodyFont = Inter({ subsets: ["latin"], variable: "--font-body" });
const headlineFont = Space_Grotesk({ subsets: ["latin"], variable: "--font-headline" });

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${bodyFont.variable} ${headlineFont.variable}`}>
      <body className="font-body">{children}</body>
    </html>
  );
}
```

Then reference `var(--font-body)` / `var(--font-headline)` from the Tailwind theme's `fontFamily` (v3) or `--font-*` (v4) so components use `font-body` / `font-headline` utilities as usual. Note `<body>` uses `font-body`, not `font-sans`: `font-sans` resolves to Tailwind's default sans stack, not to either loaded font, so applying it here would load two fonts and use neither.

**Mapping Stitch screen elements to shadcn primitives** (common cases — always prefer a primitive the project already has installed over adding a new one):

| Screen element | Likely shadcn primitive |
|---|---|
| Primary/secondary action button | `Button` (`variant="default"` / `"secondary"` / `"outline"`) |
| Text input, search field | `Input` |
| Multi-line field | `Textarea` |
| Dropdown/select control | `Select` |
| Checkbox/toggle | `Checkbox` or `Switch` |
| Card-like content block | `Card` (`CardHeader`, `CardContent`, `CardFooter`) |
| Modal/confirmation flow | `Dialog` or `AlertDialog` |
| Tab navigation | `Tabs` |
| Status pill/label | `Badge` |
| Top nav / side nav | `NavigationMenu` or a hand-rolled layout using `Sheet` for mobile |
| Data table | `Table`, or `DataTable` if the project has the TanStack Table pattern installed |
| Avatar/profile image | `Avatar` |
| Toast/inline notification | `Sonner` or `Toast`, whichever the project has installed |

If a shadcn primitive isn't installed yet and the screen clearly needs it, say so and ask before running `npx shadcn@latest add <component>` — adding a dependency/generated file is worth a quick confirmation, it isn't a pure read.

---

## Vue / Nuxt

Tailwind setup is identical to the plain Tailwind v3/v4 recipes above — Vue and Nuxt don't change how Tailwind itself is configured. Component shape differs:

```vue
<!-- components/PricingCard.vue -->
<script setup lang="ts">
interface Props {
  title: string;
  price: string;
  featured?: boolean;
}
defineProps<Props>();
</script>

<template>
  <div
    class="rounded-md border p-6 font-body"
    :class="{ 'border-primary bg-primary/5': featured }"
  >
    <h3 class="font-headline text-lg">{{ title }}</h3>
    <p class="text-2xl font-semibold">{{ price }}</p>
  </div>
</template>
```

For Nuxt specifically: Google Fonts are typically loaded via `@nuxtjs/google-fonts` (if already a dependency — check before adding it) or a plain `<link>`/`@import`, since Nuxt has no direct `next/font` equivalent. Match whichever mechanism the project already uses; only add `@nuxtjs/google-fonts` if nothing else is present and the user confirms adding a dependency.

---

## Svelte / SvelteKit

```svelte
<!-- lib/components/PricingCard.svelte — Svelte 5 runes syntax; use `export let` if the project is on Svelte 4 -->
<script lang="ts">
  let { title, price, featured = false }: {
    title: string; price: string; featured?: boolean;
  } = $props();
</script>

<div class="rounded-md border p-6 font-body {featured ? 'border-primary bg-primary/5' : ''}">
  <h3 class="font-headline text-lg">{title}</h3>
  <p class="text-2xl font-semibold">{price}</p>
</div>
```

A Tailwind class containing `/` cannot go through a `class:` directive — use a template expression as above, or the project's existing `clsx`/`cn` helper.

Tailwind configuration is the same v3/v4 recipe as above (SvelteKit's Tailwind integration doesn't change the theme layer). Fonts load via a `<link>` in `app.html` or an `@import` in the global stylesheet — there is no SvelteKit-specific font-loading API equivalent to `next/font`.

---

## Plain CSS custom properties (no framework detected)

Fallback recipe when Step 1 finds no Tailwind, no CSS-in-JS, and no component framework theming convention — just plain CSS.

```css
:root {
  --color-primary: #4f46e5;
  --color-primary-foreground: #ffffff;
  --color-secondary: #eef2ff;
  --font-headline: "Space Grotesk", sans-serif;
  --font-body: "Inter", sans-serif;
  --radius-md: 12px;
}

.pricing-card {
  border-radius: var(--radius-md);
  font-family: var(--font-body);
}

.pricing-card__title {
  font-family: var(--font-headline);
}

.pricing-card--featured {
  border-color: var(--color-primary);
}
```

For `colorMode: DARK` support, add a `[data-theme="dark"]` or `.dark` block re-declaring the same custom properties, matching whatever toggle mechanism (if any) the project already uses, or `@media (prefers-color-scheme: dark) { :root { ... } }` if the project has no explicit toggle.

---

## Font-loading table

**Read the resolved family name first.** `get_project`'s `designTheme` returns `bodyFontFamily`, `headlineFontFamily`, and `labelFontFamily` as real CSS family names (observed values include `"Inter"` and `"Newsreader"`) — use these directly, they need no further derivation.

**Fallback — derive the family name from the enum only when a resolved `*FontFamily` field is missing.** Stitch's font enum values (`headlineFont` / `bodyFont` / `labelFont`, e.g. `INTER`, `SPACE_GROTESK`) map to real Google Fonts family names. The table below covers the 12 faces most likely to appear. For any enum not in this table, derive the family name mechanically: replace underscores with spaces and title-case each word (`SPLINE_SANS` → "Spline Sans", `HANKEN_GROTESK` → "Hanken Grotesk"). Known exceptions that do not title-case cleanly: `EB_GARAMOND` → "EB Garamond", `IBM_PLEX_SANS` → "IBM Plex Sans", `IBM_PLEX_SERIF` → "IBM Plex Serif", `DM_SANS` → "DM Sans", `SOURCE_SERIF_4` → "Source Serif 4", `SOURCE_SANS_3` → "Source Sans 3". For the `next/font/google` import name, use the same family name with spaces replaced by underscores (`Plus Jakarta Sans` → `Plus_Jakarta_Sans`). If the `stitch-design-system` skill is also installed, its enum reference has the authoritative 68-value list — but do not block on it; the derivation above is sufficient.

| Stitch enum | Google Fonts family | `next/font/google` import | `<link>` snippet | CSS `@import` |
|---|---|---|---|---|
| `INTER` | Inter | `import { Inter } from "next/font/google"` | `<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">` | `@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');` |
| `PLUS_JAKARTA_SANS` | Plus Jakarta Sans | `import { Plus_Jakarta_Sans } from "next/font/google"` | `<link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700&display=swap" rel="stylesheet">` | `@import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700&display=swap');` |
| `SPACE_GROTESK` | Space Grotesk | `import { Space_Grotesk } from "next/font/google"` | `<link href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;500;600;700&display=swap" rel="stylesheet">` | `@import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;500;600;700&display=swap');` |
| `DM_SANS` | DM Sans | `import { DM_Sans } from "next/font/google"` | `<link href="https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&display=swap" rel="stylesheet">` | `@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&display=swap');` |
| `MANROPE` | Manrope | `import { Manrope } from "next/font/google"` | `<link href="https://fonts.googleapis.com/css2?family=Manrope:wght@400;500;600;700&display=swap" rel="stylesheet">` | `@import url('https://fonts.googleapis.com/css2?family=Manrope:wght@400;500;600;700&display=swap');` |
| `WORK_SANS` | Work Sans | `import { Work_Sans } from "next/font/google"` | `<link href="https://fonts.googleapis.com/css2?family=Work+Sans:wght@400;500;600;700&display=swap" rel="stylesheet">` | `@import url('https://fonts.googleapis.com/css2?family=Work+Sans:wght@400;500;600;700&display=swap');` |
| `LEXEND` | Lexend | `import { Lexend } from "next/font/google"` | `<link href="https://fonts.googleapis.com/css2?family=Lexend:wght@400;500;600;700&display=swap" rel="stylesheet">` | `@import url('https://fonts.googleapis.com/css2?family=Lexend:wght@400;500;600;700&display=swap');` |
| `OUTFIT` | Outfit | `import { Outfit } from "next/font/google"` | `<link href="https://fonts.googleapis.com/css2?family=Outfit:wght@400;500;600;700&display=swap" rel="stylesheet">` | `@import url('https://fonts.googleapis.com/css2?family=Outfit:wght@400;500;600;700&display=swap');` |
| `GEIST` | Geist | `import { Geist } from "next/font/google"` | `<link href="https://fonts.googleapis.com/css2?family=Geist:wght@400;500;600;700&display=swap" rel="stylesheet">` | `@import url('https://fonts.googleapis.com/css2?family=Geist:wght@400;500;600;700&display=swap');` |
| `SORA` | Sora | `import { Sora } from "next/font/google"` | `<link href="https://fonts.googleapis.com/css2?family=Sora:wght@400;500;600;700&display=swap" rel="stylesheet">` | `@import url('https://fonts.googleapis.com/css2?family=Sora:wght@400;500;600;700&display=swap');` |
| `PLAYFAIR_DISPLAY` | Playfair Display | `import { Playfair_Display } from "next/font/google"` | `<link href="https://fonts.googleapis.com/css2?family=Playfair+Display:wght@400;500;600;700&display=swap" rel="stylesheet">` | `@import url('https://fonts.googleapis.com/css2?family=Playfair+Display:wght@400;500;600;700&display=swap');` |
| `JETBRAINS_MONO` | JetBrains Mono | `import { JetBrains_Mono } from "next/font/google"` | `<link href="https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;500;600;700&display=swap" rel="stylesheet">` | `@import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;500;600;700&display=swap');` |

Notes:
- `next/font/google` import names replace spaces in the family name with underscores (`Plus Jakarta Sans` → `Plus_Jakarta_Sans`).
- `Geist` is a Vercel-authored font; confirm it resolves under `next/font/google` in the installed Next.js version before relying on it — in some setups it ships instead as a standalone `geist` npm package (`import { GeistSans } from "geist/font/sans"`). Check `package.json` for a `geist` dependency and prefer that import path if present.
- Always pass a `weight`/`subsets` array matching what the design actually uses rather than copying the `wght@400;500;600;700` range verbatim — trim it to the weights the screen's typography map actually references.
- Only load the faces the design system actually specifies (`headlineFont`, `bodyFont`, and `labelFont` if distinct) — don't add extra weights or families speculatively.

---

## Resolved tokens from `designTheme`: `namedColors` and `typography`

When `get_project`'s `designTheme` includes `namedColors` and `typography`, convert them directly — they're already resolved values, not enums to map. `namedColors` is a flat snake_case→hex object (47 entries in a typical project); `typography` is a set of named levels, each `{fontFamily, fontSize, fontWeight, letterSpacing, lineHeight}`. Worked example using real observed values:

```json
"namedColors": {
  "background": "#f7f9fb",
  "error": "#ba1a1a",
  "error_container": "#ffdad6",
  "inverse_on_surface": "#eff1f3"
},
"typography": {
  "body-lg": {
    "fontFamily": "Newsreader",
    "fontSize": "20px",
    "fontWeight": "400",
    "letterSpacing": "-0.005em",
    "lineHeight": "32px"
  }
}
```

**(a) Tailwind v4 `@theme` entries** — snake_case keys become kebab-case `--color-*` variables; a typography level becomes a matching set of `--text-*` / `--font-*` entries:

```css
@theme {
  --color-background: #f7f9fb;
  --color-error: #ba1a1a;
  --color-error-container: #ffdad6;
  --color-inverse-on-surface: #eff1f3;

  --font-body-lg: "Newsreader", serif;
  --text-body-lg: 20px;
  --text-body-lg--line-height: 32px;
  --text-body-lg--font-weight: 400;
  --text-body-lg--letter-spacing: -0.005em;
}
```

**(b) Plain CSS custom properties** — same values, no Tailwind-specific naming convention required:

```css
:root {
  --color-background: #f7f9fb;
  --color-error: #ba1a1a;
  --color-error-container: #ffdad6;
  --color-inverse-on-surface: #eff1f3;

  --font-body-lg: "Newsreader", serif;
  --size-body-lg: 20px;
  --line-height-body-lg: 32px;
  --weight-body-lg: 400;
  --tracking-body-lg: -0.005em;
}
```

Apply the same pattern to every `namedColors` entry and every `typography` level actually used by the screen being translated — don't pre-generate tokens for values nothing references.

---

## `roundness` → radius scale

`roundness` selects a **four-step border-radius scale**, not a single value. Each generated screen embeds `<script id="tailwind-config">tailwind.config={...}</script>`, and `theme.extend.borderRadius` inside it is the authoritative, already-resolved scale for that project — **read it from the screen when one is available.** The table below is the fallback for when no generated screen is at hand; it was measured 2026-09-16 from `theme.extend.borderRadius` on one screen from each of three real Stitch projects whose `roundness` differed:

| `roundness` | `DEFAULT` | `lg` | `xl` | `full` |
|---|---|---|---|---|
| `ROUND_FOUR` | 0.125rem (2px) | 0.25rem (4px) | 0.5rem (8px) | 0.75rem (12px) |
| `ROUND_EIGHT` | 0.25rem (4px) | 0.5rem (8px) | 0.75rem (12px) | 9999px |
| `ROUND_TWELVE` | 0.5rem (8px) | 1rem (16px) | 1.5rem (24px) | 9999px |
| `ROUND_FULL` | not observed — no sample project used it; read the screen's own `tailwind-config` instead of guessing | | | |
| `ROUND_TWO` | — | — | — | **Deprecated.** Treat as `ROUND_FOUR` and flag the substitution. |

Three things to watch for:

- **The enum name matches the `lg` step, not a global radius.** `ROUND_FOUR` → `lg` is 4px; `ROUND_EIGHT` → `lg` is 8px.
- **`ROUND_TWELVE` does not produce a 12px radius.** Its `lg` is 16px; 12px appears only as `ROUND_EIGHT`'s `xl`.
- **`rounded-full` is not always a pill.** Under `ROUND_FOUR` it resolves to 12px, not `9999px`. Only `ROUND_EIGHT` and `ROUND_TWELVE` make `full` a true pill in this sample.

## `colorVariant` → palette-generation strategy

`colorVariant` describes how far the generated secondary/tertiary palette should drift from `customColor`, in the vocabulary of Material Design 3's dynamic color system — it is not itself a formula you should try to reimplement in code. When translating a design system into token values, use it as guidance for how much contrast/hue variation to expect between primary, secondary, and tertiary tokens, not as a value you compute from:

| Value | Practical read |
|---|---|
| `MONOCHROME` | Secondary/tertiary tokens stay close to a desaturated version of the primary hue — near-grayscale accents. |
| `NEUTRAL` | Low-saturation, restrained secondary/tertiary tones — a muted, professional palette. |
| `TONAL_SPOT` | The balanced default — moderate hue shift between primary/secondary/tertiary, safe general-purpose choice. |
| `VIBRANT` | Higher saturation across the derived palette — more energetic accent colors. |
| `EXPRESSIVE` | Wider hue shifts between tokens than `TONAL_SPOT` — more visually distinct secondary/tertiary colors. |
| `FIDELITY` | Stays closest to the literal `customColor` seed with minimal derivation — use this signal to keep generated tokens visually close to the exact seed hex. |
| `CONTENT` | Derived to complement content/imagery rather than the UI chrome itself — treat similarly to `FIDELITY` absent other signal. |
| `RAINBOW` | Wide multi-hue spread — expect secondary/tertiary tokens in visibly different hues, not just different shades of the primary. |
| `FRUIT_SALAD` | Playful, high multi-hue variation — similar practical effect to `RAINBOW`, geared toward a less formal palette. |

In practice: the exact hex values for secondary/tertiary tokens should come from what `get_screen` / `list_design_systems` actually returns (via `overrideSecondaryColor` / `overrideTertiaryColor` when set), not from reimplementing this algorithm. Use `colorVariant` only to sanity-check that the returned tokens look consistent with the stated strategy, and to explain to the user why a palette looks the way it does if asked.
