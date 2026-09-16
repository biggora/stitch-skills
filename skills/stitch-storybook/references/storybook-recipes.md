# Storybook Recipes — Reference

These recipes target recent Storybook (8/9-era) conventions as understood as of early-to-mid 2026. **The installed version's own generated config is authoritative.** Storybook's major versions have moved fast — config shape, addon APIs, and even the CSF syntax details have changed release to release. Read `.storybook/main.*` (and `preview.*`) before editing either file, and if what you find there disagrees with anything below, follow what's actually installed and say so.

Read `SKILL.md` first. This file assumes Step 1 (stack detection) is already done and Storybook either already exists or was just scaffolded with `npx storybook@latest init`.

---

## Per-framework starting points

Each block shows the framework package Storybook's initializer installs for that stack, plus a minimal `main.*`/`preview.*` shape. Treat these as a starting silhouette to compare against what init actually generated — not a file to paste over it.

### React + Vite

Framework package: `@storybook/react-vite`

```js
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/react-vite';

const config: StorybookConfig = {
  stories: ['../src/**/*.mdx', '../src/**/*.stories.@(js|jsx|mjs|ts|tsx)'],
  addons: ['@storybook/addon-essentials'],
  framework: { name: '@storybook/react-vite', options: {} },
};
export default config;
```

```ts
// .storybook/preview.ts
import type { Preview } from '@storybook/react';
import '../src/index.css'; // the project's real global stylesheet

const preview: Preview = {
  parameters: {
    backgrounds: { default: 'surface' },
    controls: { matchers: { color: /(background|color)$/i, date: /Date$/i } },
  },
};
export default preview;
```

### Next.js

Framework package: `@storybook/nextjs` (handles `next/image`, `next/font`, `next/router` mocking, and Next's own Webpack/Turbopack config out of the box — do not hand-roll a plain React Vite config against a Next.js app).

```js
// .storybook/main.ts
const config = {
  stories: ['../app/**/*.stories.@(js|jsx|ts|tsx)', '../components/**/*.stories.@(js|jsx|ts|tsx)'],
  addons: ['@storybook/addon-essentials'],
  framework: { name: '@storybook/nextjs', options: {} },
};
export default config;
```

`preview.ts` imports the same root stylesheet the app's root layout imports (commonly `app/globals.css`).

### Vue 3

Framework package: `@storybook/vue3-vite`

```js
// .storybook/main.ts
const config = {
  stories: ['../src/**/*.stories.@(js|ts)'],
  addons: ['@storybook/addon-essentials'],
  framework: { name: '@storybook/vue3-vite', options: {} },
};
export default config;
```

```ts
// .storybook/preview.ts
import '../src/assets/main.css';
export default {
  parameters: { backgrounds: { default: 'surface' } },
};
```

### Svelte / SvelteKit

Framework package: `@storybook/svelte-vite` for plain Svelte+Vite, `@storybook/sveltekit` when the project is SvelteKit (it layers routing/module-mock support on top of the Vite one — check which one init installed).

```js
// .storybook/main.ts
const config = {
  stories: ['../src/**/*.stories.@(js|ts|svelte)'],
  addons: ['@storybook/addon-essentials'],
  framework: { name: '@storybook/sveltekit', options: {} },
};
export default config;
```

`preview.ts` imports the project's global CSS (e.g. `../src/app.css` for a Tailwind v4 SvelteKit app).

### Angular

Framework package: `@storybook/angular`

```js
// .storybook/main.ts
const config = {
  stories: ['../src/**/*.stories.@(js|ts)'],
  addons: ['@storybook/addon-essentials'],
  framework: { name: '@storybook/angular', options: {} },
};
export default config;
```

Angular's `preview.ts` typically imports global styles via `angularOptions` in `main.ts` (`styles: ['src/styles.css']`) rather than a plain `import` in `preview.ts` — check what init generated, since Angular's build pipeline differs from the Vite-based frameworks above.

---

## Worked example: `ArticleCard` through to a composed `HomePage`

One component, carried through the whole pipeline: `ArticleCard`, sliced from a Stitch blog-listing screen. Fields: `title`, `excerpt`, `author`, `date`, `image`, and a `featured` boolean that the Stitch screen shows as a larger, two-column-spanning card variant.

### Component story (React + CSF3 shown; translate `args`/`argTypes` shape to the detected framework's CSF equivalent)

```tsx
// src/components/ArticleCard/ArticleCard.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { ArticleCard } from './ArticleCard';

const meta: Meta<typeof ArticleCard> = {
  title: 'Components/ArticleCard',
  component: ArticleCard,
  tags: ['autodocs'],
  argTypes: {
    featured: { control: 'boolean' },
  },
};
export default meta;

type Story = StoryObj<typeof ArticleCard>;

const baseArgs = {
  title: 'Designing for Dark Mode Without a Second Design',
  excerpt: 'A class-based dark strategy keeps one set of components honest in both themes.',
  author: 'Priya Nandakumar',
  date: '2026-03-04',
  image: '/images/dark-mode-cover.jpg',
};

export const Default: Story = { args: { ...baseArgs, featured: false } };

export const Featured: Story = { args: { ...baseArgs, featured: true } };

export const LongTitle: Story = {
  args: {
    ...baseArgs,
    title: 'When the Design System and the Codebase Disagree About What "Primary" Means',
    featured: false,
  },
};
```

Three variant stories: the default card, the `featured` layout variant the Stitch screen actually shows, and a content-stress case (long title) — not a fourth invented state the design never depicted.

### Page composition: `HomePage`

The `HomePage` story assembles nine `ArticleCard`s plus the screen's header, nav, and footer components — the same structural counts and roles the measured Stitch screen shows — using the real components, not re-implemented markup:

```tsx
// src/pages/HomePage.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { HomePage } from './HomePage';
import { articles } from './__fixtures__/homepage-articles'; // 9 entries, real screen copy

const meta: Meta<typeof HomePage> = {
  title: 'Pages/HomePage',
  component: HomePage,
  tags: ['autodocs'],
  parameters: {
    // reference image for comparison, pulled from get_screen's screenshot.downloadUrl
    docs: { description: { story: 'Compare against docs/stitch-screens/homepage.png.' } },
  },
};
export default meta;

type Story = StoryObj<typeof HomePage>;

export const Default: Story = {
  args: { articles }, // real titles/excerpts/authors from the Stitch screen, not lorem ipsum
};
```

`HomePage` itself renders `<Header />`, `<Nav />`, a grid of `<ArticleCard featured={i === 0} />`, and `<Footer />` — the same components each already have their own story files. Nothing about the page layout is re-typed here; it is composed from what Steps 4 already produced.

---

## Tailwind v4 vs. v3 inside Storybook

Stitch's own generated screens set `darkMode: "class"` in their inline Tailwind config — dark mode is a class toggle on a root element, not a media-query-only strategy. Whichever Tailwind version the project uses, Storybook's dark-mode toggle (a `withThemeByClassName` decorator from `@storybook/addon-themes`, or an equivalent) should add/remove that same class on the preview root, not simulate `prefers-color-scheme`, so it matches what the class-based strategy in the real app actually does.

**Tailwind v3** (`tailwind.config.js`, PostCSS pipeline): Storybook's Vite/Webpack build picks up the existing PostCSS config automatically as long as `preview.ts` imports the stylesheet that starts with `@tailwind base; @tailwind components; @tailwind utilities;`. No Storybook-specific Tailwind config is normally needed beyond that import.

**Tailwind v4** (CSS-first, `@import "tailwindcss";` plus an `@theme` block, no JS config file by default): the same import in `preview.ts` is still the mechanism, but there is no `tailwind.config.js` for Storybook's build to discover implicitly in some setups — confirm the project's Vite/PostCSS config already wires up the Tailwind v4 plugin, since Storybook reuses the project's own bundler config rather than adding a separate Tailwind pipeline. If styles don't apply, the v4 plugin registration in the project's `vite.config.*` (or PostCSS config) is the first thing to check, not `preview.ts`.

In both versions, `darkMode: "class"` means the toggle only works if something actually adds a `dark` class to an ancestor of the story root — a Storybook decorator, not a CSS media query.

---

## Design-tokens docs pages

### Typography docs from `designTheme.typography`

Build a docs-only story that renders each named level (`display-xl`, `display-xl-mobile`, `headline-lg`, `headline-lg-mobile`, `headline-md`, `headline-sm`, `body-lg`, `body-md`, `body-sm`, `label-md`, `label-sm`) with its own resolved `{fontFamily, fontSize, fontWeight, letterSpacing, lineHeight}` applied inline or via a matching CSS class, and the level name printed alongside it:

```tsx
// src/docs/Typography.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';

const meta: Meta = { title: 'Design Tokens/Typography' };
export default meta;

const levels = [
  { name: 'display-xl', fontFamily: 'Newsreader', fontSize: '57px', fontWeight: '400', letterSpacing: '-0.005em', lineHeight: '64px' },
  { name: 'headline-lg', fontFamily: 'Newsreader', fontSize: '32px', fontWeight: '400', letterSpacing: '0em', lineHeight: '40px' },
  { name: 'body-md', fontFamily: 'Inter', fontSize: '14px', fontWeight: '400', letterSpacing: '0.0025em', lineHeight: '20px' },
  // ...remaining levels from the project's actual designTheme.typography
];

export const AllLevels: StoryObj = {
  render: () => (
    <div>
      {levels.map((l) => (
        <p key={l.name} style={{ fontFamily: l.fontFamily, fontSize: l.fontSize, fontWeight: l.fontWeight, letterSpacing: l.letterSpacing, lineHeight: l.lineHeight }}>
          {l.name} — The quick brown fox jumps over the lazy dog
        </p>
      ))}
    </div>
  ),
};
```

Pull the real values from the project's own `designTheme.typography` (however `stitch-design-system`/`stitch-to-code` already surfaced them) rather than retyping the example numbers above verbatim — those are illustrative, not the current project's actual tokens.

### Color palette docs from `namedColors`

`designTheme.namedColors` is roughly 47 snake_case-keyed hex entries (`background`, `surface`, `surface_container_low`, `error`, `error_container`, `on_primary`, `outline_variant`, `inverse_surface`, …). Render them as swatches grouped by prefix (`surface*`, `on_*`, `error*`, `inverse_*`, plain roles) so the docs page reads as a palette rather than an unordered dump:

```tsx
// src/docs/ColorPalette.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';

const meta: Meta = { title: 'Design Tokens/Color Palette' };
export default meta;

export const AllColors: StoryObj = {
  render: () => (
    <div style={{ display: 'grid', gridTemplateColumns: 'repeat(auto-fill, minmax(140px, 1fr))', gap: '8px' }}>
      {Object.entries(namedColors).map(([name, hex]) => (
        <div key={name}>
          <div style={{ background: hex, height: 56, borderRadius: 8 }} />
          <code>{name}</code> <span>{hex}</span>
        </div>
      ))}
    </div>
  ),
};
```

`namedColors` should come from the project's own stored `get_project` response or the tokens `stitch-design-system` already applied — don't re-fetch or hardcode a stale snapshot into the docs page.

---

## `deviceType` → Storybook viewport mapping

| Stitch `deviceType` | Storybook viewport | Notes |
|---|---|---|
| `MOBILE` | `mobile1` / a custom `{ width: '390px', height: '844px' }` entry | Match the actual measured `width`/`height` strings from `get_screen` when precision matters. |
| `TABLET` | `tablet` | Storybook's default `tablet` preset (768×1024) is a reasonable default; override if the screen's own `width`/`height` differ meaningfully. |
| `DESKTOP` | `responsive` at a wide default, or a custom `{ width: '1440px', height: '900px' }` entry | Desktop screens vary more; prefer a custom entry matching the screen's real `width`. |
| `AGNOSTIC` | No forced viewport (leave Storybook's default) | The design has no fixed device target — don't force a viewport that implies one. |

`get_screen` returns `width`/`height` as strings — parse them before using them as viewport numbers.

---

## Troubleshooting

**Styles not applying in stories.** Almost always a missing or wrong stylesheet import in `.storybook/preview.*` — confirm it imports the same root CSS file the app's real entry point imports, not a stale or partial one. For Tailwind v4, also confirm the bundler-level Tailwind plugin is registered in the config Storybook actually uses (Storybook reuses the project's Vite/PostCSS config; it doesn't get Tailwind for free just because the project has it).

**Fonts missing.** If fonts load via `next/font` or a similar build-time mechanism, that mechanism may not run inside Storybook's own build unless the matching framework package (`@storybook/nextjs`) is in use. Otherwise, load the `bodyFontFamily`/`headlineFontFamily` fonts via a `<link>` or `@import` inside `preview.ts`'s imported stylesheet, the same way any other global asset loads for stories.

**Dark mode not toggling.** Confirm the toggle actually adds/removes a class on an ancestor element (matching Stitch's `darkMode: "class"` strategy) rather than trying to flip `prefers-color-scheme`, which most browsers and Storybook's own preview iframe won't let a toggle button override.

**Path alias resolution failing** (e.g. `@/components/...` resolves in the app but not in stories). Storybook's Vite/Webpack build needs the same alias config as the app — check whether `.storybook/main.*` extends the project's `vite.config.*`/`tsconfig.json` paths, or needs an explicit `viteFinal`/`webpackFinal` hook adding the same resolve alias.
