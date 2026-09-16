---
name: stitch-storybook
description: Sets up Storybook, adds components to Storybook, creates a Storybook story, writes stories for components extracted from a Google Stitch screen, builds Storybook pages that compose Stitch components into full-screen layouts, wires a Stitch design system's colors and typography into Storybook's theme and backgrounds, configures Storybook viewports to match a Stitch screen's deviceType, and documents a design system in Storybook via a design-tokens docs page. Applies whenever the user asks to scaffold Storybook, register a component as a story, build a Storybook page from Stitch-derived components, add dark mode or backgrounds to Storybook matching Stitch's theme, or otherwise wants Storybook coverage for components that came from Google Stitch — and also applies to plain "set up Storybook" or "add a story for this component" requests in any project where Stitch-derived components already exist, even without Stitch named explicitly.
---

# Stitch Storybook

Owns the path from components — however they got into the project, most often extracted from a Google Stitch screen by `stitch-components` (or equivalent manual extraction) — to a working Storybook: scaffolding it if absent, wiring the project's design tokens into it so stories actually look like the design, writing stories per component, and composing page-level stories that reassemble those components into the original Stitch screen layout for side-by-side comparison. This skill does not generate Stitch screens, extract components from them, or apply design systems inside Stitch itself — for that, use `stitch-screens`, `stitch-components`, and `stitch-design-system` if installed, or call the underlying `mcp__stitch__*` tools directly (`get_screen`, `get_project`, `list_screens`).

**The skill is stack-agnostic.** Storybook runs on React, Vue, Svelte, Angular, and plain web components, with a different framework package and config shape for each. Never assume React. Detect the host project's real stack before touching any Storybook file.

Work through the six steps below in order. Once the stack is known (end of Step 1), **read `references/storybook-recipes.md`** for the concrete, copy-ready per-framework recipes, the worked `ArticleCard`/`HomePage` example, and the Tailwind v4-vs-v3 and troubleshooting notes — do not improvise `.storybook/main.*` or `preview.*` contents from memory when a checked recipe exists for the detected stack.

## Step 1 — Detect

Before writing or running anything, determine what already exists:

1. **Check for an existing Storybook.** Look at `package.json` for `storybook` or any `@storybook/*` dependency, and look for a `.storybook/` directory at the project root. If either is present, Storybook is already installed.
2. **If Storybook already exists, read its actual config** — `.storybook/main.*` and `.storybook/preview.*` — before changing anything. **Extend what is there; never re-run the initializer over a configured Storybook.** Re-initializing is destructive: it can overwrite addon lists, framework config, and preview decorators the project already depends on.
3. **If no Storybook exists, detect the framework** from `package.json` dependencies: `react` (and separately, `next` for Next.js), `vue`/`nuxt`, `svelte`/`@sveltejs/kit`, `@angular/core`, or a plain web-components setup with no framework dependency at all.
4. **Detect the bundler** — Vite (`vite` dependency, `vite.config.*`), Webpack, or a Next.js build — since it changes which framework package Storybook needs.
5. **Detect TypeScript** via `tsconfig.json`, and **detect the styling system**: Tailwind (check the `tailwindcss` version — v3 uses a JS config, v4 is CSS-first with `@theme`), CSS Modules, or plain CSS. This mirrors the detection `stitch-to-code` already does when translating a screen to code — if that skill's output is present in the project, its conclusions about stack and styling are already correct and don't need re-deriving.

## Step 2 — Scaffold, if needed

If Step 1 found no Storybook, the current initializer is:

```
npx storybook@latest init
```

It detects the framework itself and installs the matching framework package — `@storybook/react-vite`, `@storybook/vue3-vite`, `@storybook/svelte-vite`, `@storybook/nextjs`, or `@storybook/angular`, among others. **Confirm with the user before running it.** It writes `.storybook/` config, edits `package.json` (scripts and dependencies), and installs packages — all things the user should approve first, not discover after the fact.

After it runs, **read what it actually produced** rather than assuming a config shape. Storybook's major versions have moved fast and its generated `main.*`/`preview.*` files differ across versions and framework packages; the installed version's own generated config is authoritative over anything described in this skill or its reference file. Report the framework package and Storybook version that were actually installed.

## Step 3 — Wire the design system into Storybook

This is the step that makes stories look like the Stitch design instead of unstyled markup:

- **Import the project's global stylesheet** in `.storybook/preview.*` so Tailwind (or whatever styling system Step 1 found) applies inside every story, the same way it applies in the real app.
- **Map the Stitch theme into Storybook's controls.** If the project has design tokens available — `designTheme.namedColors` from `get_project`, or tokens already translated into the project by `stitch-design-system`/`stitch-to-code` — turn `namedColors` into named background options, and add a dark-mode toggle that matches the `darkMode: "class"` strategy Stitch's own generated HTML uses (a `.dark` class on a root element, not a media-query-only approach), so the toggle actually reflects how the project's dark mode works.
- **Add viewport presets matching the screens' `deviceType`.** A Stitch `get_screen` response's `deviceType` is one of `MOBILE`, `DESKTOP`, `TABLET`, `AGNOSTIC`; map each to a Storybook viewport entry (or Storybook's default viewport addon presets) so a component pulled from a mobile screen previews at a mobile width by default. `references/storybook-recipes.md` has the mapping table.
- **Load the fonts named by `bodyFontFamily` / `headlineFontFamily`** from `designTheme` (or from the project's own font-loading mechanism, if `stitch-to-code` already wired them in) so story typography matches the design instead of falling back to a browser default.
- **Never add the Tailwind CDN script to a real project.** Stitch's own generated screen HTML loads `https://cdn.tailwindcss.com` for preview purposes — that is a Stitch-side convenience, not something to carry into Storybook. The project must have Tailwind (or its real styling system) actually installed and built; importing the project's own compiled/processed CSS in `preview.*` is correct, adding the CDN `<script>` tag is not.

## Step 4 — Write stories for components

One story file per component, colocated the way the project already colocates things (next to the component, or in a parallel `stories/` tree — match what's there). For each component:

- A **default story** showing its baseline state.
- **One story per meaningful variant** the Stitch design actually shows (e.g., a `featured` boolean, a size or emphasis variant) — don't invent variants the design doesn't have.
- **Stories for loading, empty, error, or disabled states**, but only where the design actually shows that state. Do not fabricate a loading spinner story for a component whose Stitch screen never depicts one.
- Use `args` for the component's props and `argTypes` for the controls panel, and prefer CSF3 object-export syntax (`export const Default: Story = { args: {...} }`) as the current convention — but confirm against the installed Storybook version, since CSF2 vs. CSF3 and the exact `Meta`/`StoryObj` typing differ by major version.
- Enable autodocs (typically a `tags: ['autodocs']` entry on the default export) so each component gets a generated docs page, matching whatever the installed version's docs mechanism actually is.

## Step 5 — Compose pages from components

This is the point of the whole pipeline: a page-level story assembles the already-written component stories into the original Stitch screen's layout, so the screen and the implementation can be compared side by side.

- **Compose at the page level; don't duplicate component internals.** A `HomePage` story imports and arranges the real `ArticleCard`, `Header`, `Nav` components — it does not re-implement their markup inline.
- **Supply realistic data taken from the screen's actual content** — the titles, labels, and copy visible in the Stitch screen or returned by `get_screen` — not lorem ipsum. This is what makes the page story a meaningful comparison target rather than generic filler.
- **Use the screenshot from `screenshot.downloadUrl`** (returned by `get_screen`) as the visual reference when composing the layout, and mention it in the story's documentation so reviewers know what it's being compared against.
- **This composition is the natural place to hang visual-regression or accessibility checks** (a Storybook test-runner integration, an a11y addon panel) if the project has them configured — but don't claim either is wired up unless you've confirmed it's actually installed and configured; say what's present and what isn't rather than assuming a default.

## Step 6 — Verify

Run the project's real Storybook build or dev script (check `package.json` for `storybook`/`build-storybook` or the current equivalents) and report the actual result. Do not claim a story renders correctly without having built or run it. If the build fails, report the failure and its output — do not describe the work as complete because the files were written.

## Honesty rules

- **Never claim a story renders correctly without having built or run it.** "I wrote the story" and "the story renders" are different claims — only make the second after Step 6 actually passes.
- **Never invent a Storybook addon API you're unsure of.** Addon APIs (viewport, backgrounds, a11y, test-runner) change across major versions. If you're not certain an API surface is correct for the installed version, say the installed version needs to be checked rather than presenting a guess as fact.
- **Never overwrite an existing `.storybook/` config without showing the user the diff first.** Extending an existing config is expected; silently replacing it is not.
- **Report the actually-installed Storybook version and framework package**, not the one you assumed going in — Step 1 and Step 2 exist precisely so the rest of the work is grounded in what's really there.
