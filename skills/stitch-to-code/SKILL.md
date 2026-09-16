---
name: stitch-to-code
description: Translates a Google Stitch design — a generated screen plus its design system — into working application code in the user's own project, matching that project's existing framework, styling system, and component conventions. Applies when the user says things like "turn this Stitch screen into code", "implement this Stitch design in my app", "export the Stitch mockup to React", "build this Stitch screen as a Next.js page", "apply the Stitch design system to my Tailwind config", "convert the Stitch design into shadcn components", or "code up the screen we just generated in Stitch", or otherwise asks for handoff from a Stitch screen or design system to production code in React, Next.js, Vue, Nuxt, Svelte, Angular, or plain CSS.
---

# Stitch to Code

Owns the handoff from a Google Stitch design (a generated screen plus its design system) to real, working code in the user's own project, via the Stitch MCP server (`stitch`, tools exposed as `mcp__stitch__<tool>`). This skill does not generate or edit Stitch screens — for that, use `stitch-screens` or `stitch-iterate` if installed, or call `generate_screen_from_text` / `edit_screens` directly. This skill starts once a screen already exists in a Stitch project and the goal is to produce code from it.

**The core discipline of this skill is stack-agnostic translation.** A Stitch screen is a visual artifact. The code you produce must look like it was written by the same team that wrote the rest of the target project — same framework, same styling approach, same component conventions, same file layout. Never impose a preferred stack. Detect the real one first, then translate into it.

Work through the five steps below, in order. Do not skip Step 1 to save time — every later step depends on its output.

## Step 1 — Detect the stack before writing a single line

Before touching the Stitch MCP tools, read the target project to determine what you're generating code into. Do this even if the user names a framework — their description is often incomplete (they say "React" and mean "Next.js App Router with shadcn/ui and Tailwind v4"), and only reading the files gives you the details that matter for matching conventions.

1. **Read `package.json`** at the consuming project root supplied by the host, or the working directory the user is already in. Do not require a host-specific environment variable. Check `dependencies` and `devDependencies` for:
   - `react`, `next` → React or Next.js (check for `next` specifically; a project can have `react` without `next`)
   - `vue`, `nuxt` → Vue or Nuxt
   - `svelte`, `@sveltejs/kit` → Svelte or SvelteKit
   - `@angular/core` → Angular
   - `solid-js` → Solid
   - `tailwindcss` and its version (v3 vs v4 changes the recipe — see Step 3)
   - `styled-components`, `@emotion/react`, `@emotion/styled` → CSS-in-JS
   - `clsx`, `classnames`, or a local `cn` re-export of either → the project's class-composition helper
2. **Look for `tailwind.config.js` / `tailwind.config.ts` / `tailwind.config.cjs`.** Its presence plus a v3-range `tailwindcss` dependency means classic JS-config Tailwind. Its *absence* alongside a v4-range `tailwindcss` dependency, combined with an `@import "tailwindcss";` line and an `@theme { ... }` block in a root CSS file (commonly `app/globals.css`, `src/index.css`), means Tailwind v4's CSS-first config. Read whichever one exists — don't assume a version from the dependency string alone if the lockfile pins something different.
3. **Look for `components.json`** at the project root. Its presence means shadcn/ui is installed; read it for the configured `style`, `baseColor`, `cssVariables`, and aliases (`components`, `utils`) so generated components land in the right directories and import from the right paths.
4. **Detect the styling approach** if Tailwind isn't present: CSS Modules (`*.module.css` files next to components), styled-components/emotion (imports of those packages inside component files), or plain CSS/SCSS (global stylesheets, BEM-ish class names). Don't assume Tailwind by default — plenty of real projects use none of it.
5. **Detect TypeScript** via a `tsconfig.json` at the root. If present, all generated code is TypeScript; if absent, plain JS/JSX matching the project's existing extensions.
6. **Read 2–3 existing components** (not boilerplate scaffolding — actual feature components if you can find them) to learn conventions no config file tells you directly:
   - File naming (`PascalCase.tsx`, `kebab-case.tsx`, `index.tsx` per folder)
   - Export style (default export vs. named export)
   - Props typing style (inline type literal, a separate `interface`/`type` above the component, destructured vs. `props.x`)
   - The class-composition helper actually in use and how it's imported (`import { cn } from "@/lib/utils"` is the shadcn convention, but confirm it against what the project really does)
   - Directory layout (`components/ui` vs `components/` vs feature-colocated components, where pages/routes live)

**Match what you find. Do not "improve" it.** If the project uses default exports and inline prop types, your new components do too, even if you'd personally write it differently. Consistency with the existing codebase outranks any general best practice you might otherwise reach for.

**If no frontend stack is detected at all** — no recognizable framework dependency, no styling system, nothing to read conventions from — stop and ask the user what to target rather than guessing or defaulting to a stack of your own choosing.

Once the stack is identified, **read `references/stack-recipes.md`** for the concrete, copy-ready recipe that matches it. Do not improvise Tailwind v4 `@theme` syntax, shadcn CSS-variable names, or `next/font/google` usage from memory when a checked recipe exists for exactly that combination.

## Step 2 — Pull the design

With the target stack known, pull the actual design content from Stitch:

1. **`list_screens`** with `projectId` (bare id, no `projects/` prefix) to find the screen(s) in scope, if you don't already have the specific screen id.
2. **`get_screen`** with `name` in the form `projects/{project}/screens/{screen}` — **both** segments prefixed, unlike `list_screens`. This returns the screen's actual content: the layout and elements you're translating into code. Get every screen in scope before writing components, not one at a time interleaved with coding — you want the full picture before decomposing (Step 4).
3. **`list_design_systems`** with `projectId` (bare, optional but treat as required in practice — omitting it lists **global** design systems only, and the global and project-scoped sets never overlap, so omitting it silently returns the wrong set) to retrieve the theme tokens governing color, typography, and shape.

**The design system is the source of truth for color, typography, and radius — not the screen.** A screen's rendered content may show a button in a particular shade of blue, but that shade exists because the design system's `customColor`/`overridePrimaryColor` resolves to it, not because blue is a property of that one button. Express colors, fonts, and corner radii through the tokens established in Step 3, not as literals copied from a screen's rendering. If a screen visibly deviates from its own project's design system, flag that to the user rather than silently encoding the deviation as a new hardcoded value.

If `get_project` (`name` in the form `projects/{id}`, **with** the prefix) is useful for resolving screen instance ids or confirming which design system a project has applied, use it — but `list_screens` and `list_design_systems` cover the common path.

## Step 3 — Translate tokens first, components second

Before writing a single component, establish the design system's tokens in the project's own token mechanism: a Tailwind `@theme` block (v4) or `tailwind.config.ts` theme extension (v3), a set of CSS custom properties, or a plain design-token module — whichever Step 1 identified. Components then reference those tokens (a class name, a CSS variable, an imported constant) instead of embedding literal hex codes, pixel values, or font names inline. This ordering matters: writing components against literals first and "tokenizing later" almost never actually happens, and it's how design systems drift out of sync with their own screens.

Apply these mapping rules, detailed further with worked examples in `references/stack-recipes.md`:

- **`roundness`** → a radius scale: `ROUND_FOUR` → `4px`, `ROUND_EIGHT` → `8px`, `ROUND_TWELVE` → `12px`, `ROUND_FULL` → `9999px` (pill/circular). `ROUND_TWO` is deprecated in the Stitch API; if you encounter it, treat it the same as `ROUND_FOUR` and note the substitution to the user.
- **`headlineFont` / `bodyFont` / `labelFont`** (Google Fonts enum names, e.g. `INTER`, `SPACE_GROTESK`) → the actual Google Fonts family name, loaded through whatever mechanism is idiomatic for the stack: `next/font/google` for Next.js, a `<link>` tag for plain HTML/CSS, an `@import` in the stylesheet otherwise. Never hardcode a font family name as a raw string scattered across components — bind it to a token exactly once, at the same layer as the other tokens.
- **`customColor`** plus any of `overridePrimaryColor` / `overrideSecondaryColor` / `overrideTertiaryColor` / `overrideNeutralColor` → the project's color token slots (Tailwind theme colors, shadcn's `--primary`/`--secondary`/etc. CSS variables, or a plain `--color-*` custom property set). An override, when present, wins over a value derived from `customColor` for that slot.
- **`colorMode`** (`LIGHT` | `DARK`) → the project's light/dark strategy. If the project already has a dark-mode mechanism (a `.dark` class toggle, `prefers-color-scheme` query, a theme provider), use it as-is; don't introduce a second, competing one. If `colorMode` is `DARK` and the project has no dark-mode support at all, say so rather than silently forcing dark-only styles into a light-only project.
- **`spacing`** (name → value) → the corresponding spacing scale for the stack (Tailwind theme spacing keys, CSS custom properties, or a spacing constants module).
- **`typography`** (level name like `display-lg` / `body-md` → `{fontFamily, fontSize, fontWeight, letterSpacing, lineHeight}` as CSS value strings) → typographic utility classes, a CSS custom-property-driven type scale, or a component-level style map, matching how the project already handles type scales if it has one.

## Step 4 — Decompose the screen

Translate the screen content pulled in Step 2 into components at the right granularity:

- **Identify repeated elements** (card patterns, list rows, form field groups, button variants used more than once) and lift them into shared components rather than duplicating markup for each occurrence.
- **Keep page-level composition in the route/page file** (the Next.js route's `page.tsx`, the Vue route's page component) — that file assembles shared components into the screen's layout; it shouldn't itself contain deeply nested one-off markup that belongs in a component.
- **Prefer the project's existing primitives over new ones.** If the project already has a `Button`, `Card`, or `Input` component — hand-rolled or from shadcn/ui — use it instead of writing a parallel one-off version. Read enough of the existing component library (found during Step 1) to know what's already available before duplicating it.
- Name new components and files consistently with what Step 1 found, not a naming scheme of your own preference.

## Step 5 — Verify

Run whatever the project actually provides for typechecking, linting, and building — check `package.json` scripts (`typecheck`, `lint`, `build`, or their project-specific equivalents) and the presence of `tsconfig.json` for a raw `tsc --noEmit` fallback. Do not invent a verification command the project doesn't have, and do not skip verification because "it looks right." State plainly, without hedging into vagueness, what actually passed and what did not — including partial results (e.g. "lint passes; the build was not run because no build script exists").

## Honesty rules

- **A Stitch screen is a design, not a specification of behavior.** It shows what a button looks like, not what happens when it's clicked. Do not invent business logic, API calls, validation rules, or routes the user didn't ask for and the screen cannot itself specify. Where a screen visibly implies behavior — a "Submit" button, a search field, a tab that presumably switches content — implement the static UI faithfully and flag the implied behavior as a decision point rather than guessing at an implementation.
- **Flag every place the design implies behavior that must be decided**, explicitly, in your response to the user — don't bury it in a code comment they may not read.
- **Do not claim visual fidelity you have not verified.** You are translating structure and tokens into code, not rendering the result and comparing it pixel-for-pixel to the Stitch screen unless you actually have a way to do that (e.g. a running dev server you can inspect). Say what you translated and on what basis, not "this matches the design exactly."
- If a value doesn't map cleanly onto anything in `references/stack-recipes.md` or the project's conventions, say so and propose the closest reasonable mapping rather than silently picking one.
