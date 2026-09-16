---
name: stitch-components
description: Slices a generated Google Stitch screen's downloaded HTML into reusable framework components in the user's own project. Applies when the user says things like "slice this Stitch screen into components", "extract components from a Stitch design", "break this Stitch screen into reusable parts", "componentize the Stitch mockup", "pull the HTML out of Stitch", "turn the Stitch screen into a component library", "split this Stitch page into a header, card grid, and footer", "find the repeated blocks in this Stitch screen", or otherwise asks to decompose a Stitch-generated screen's markup into named, reusable components rather than one flat page. Covers fetching the screen's real HTML via `get_screen`'s `htmlCode.downloadUrl`, proposing a slicing plan for the user's approval, and only then extracting components matched to the host project's own framework and styling conventions.
---

# Stitch Components

Owns slicing a Google Stitch-generated screen's actual HTML into reusable components, via the Stitch MCP server (`stitch`, tools exposed as `mcp__stitch__<tool>`). This skill does not generate or edit Stitch screens — for that, use `stitch-screens` or `stitch-iterate` if installed, or call `generate_screen_from_text` / `edit_screens` directly. It also does not do the general design-to-code translation that turns a whole screen into a page — for that, use `stitch-to-code` if installed. This skill's job is narrower and comes first: look at one screen's real markup, find the seams where it repeats or separates into landmarks, propose a component breakdown, get it approved, and only then write files. If `stitch-storybook` is installed, hand the extracted components to it afterward for story generation; if not, that step is simply skipped — nothing here depends on it.

**The defining behavior of this skill is propose-then-write.** Steps 1–3 below produce a plan and nothing else. Do not create or overwrite a single component file until the user has explicitly approved that plan. This is not a formality — a wrong slicing decision compounds across every component built on top of it, and it is far cheaper to correct a table than to redo files.

Work through the five steps below, in order.

## Step 1 — Fetch the markup

`get_screen` does **not** return HTML directly. Its response carries an `htmlCode` object whose `downloadUrl` is a separate signed URL you must fetch yourself:

1. If you don't already have the target screen's id, call **`list_screens`** with `projectId` (bare id, no `projects/` prefix) to find it.
2. Call **`get_screen`** with `name` in the form `projects/{project}/screens/{screen}` — **both** segments prefixed. The response looks like:

   ```json
   {
     "name": "projects/16257537201871244185/screens/000eb28cacb04bbda54fedfc44ba7525",
     "title": "...",
     "deviceType": "DESKTOP",
     "width": "2616",
     "height": "5548",
     "htmlCode": {
       "name": "projects/.../screens/.../fileEntries/html",
       "mimeType": "text/html",
       "downloadUrl": "https://contribution.usercontent.google.com/download?c=..."
     },
     "screenshot": {
       "name": ".../fileEntries/screenshot",
       "downloadUrl": "https://lh3.googleusercontent.com/aida/..."
     }
   }
   ```

   Note `width` and `height` come back as **strings**, not numbers — don't assume a numeric type when using them.
3. Download the actual markup from `htmlCode.downloadUrl`: `curl -sL -o <file> "<url>"`. This URL needs no authentication — a plain `curl -sL` succeeds. Save it into a scratch or temp directory, never into the user's source tree; it's raw input to this skill's analysis, not a deliverable.
4. **Treat the download URL as expiring.** It's a signed Google link. If time has passed since you called `get_screen` — a prior turn, a long conversation, a retried step — re-call `get_screen` for a fresh `downloadUrl` rather than reusing one you fetched earlier. Do not assume a URL you saw a few tool calls ago is still valid.
5. When a visual reference will help (judging spacing, confirming a repeated card's visual identity, checking a layout you're unsure how to read from markup alone), also fetch `screenshot.downloadUrl` the same way. **State plainly that you cannot see an image unless you actually read it** — don't reason about visual layout from the screen's title or markup structure alone and imply you looked at a screenshot when you didn't.

## Step 2 — Detect the host project's conventions before deciding anything

Do this before proposing any component. A slicing plan that assumes the wrong stack has to be redone, not adjusted.

1. Read `package.json` for the framework (`react`, `next`, `vue`, `nuxt`, `svelte`, `@angular/core`, `solid-js`) and styling system (`tailwindcss` and its version, `styled-components`, `@emotion/react`, `clsx`/`classnames` or a local `cn` re-export).
2. Look for `tailwind.config.js/.ts/.cjs` (classic v3 JS config) versus an `@import "tailwindcss"` line plus a `@theme { ... }` block in a root CSS file (v4 CSS-first config) — their presence or absence tells you which Tailwind generation the project is actually on, not the dependency version string alone.
3. Look for `components.json` at the project root — its presence means shadcn/ui is installed; read its `style`, `baseColor`, `cssVariables`, and `aliases` so new components land in the right directories and import from the right paths.
4. Check for `tsconfig.json` to decide TypeScript vs. plain JS/JSX.
5. Read 2–3 existing components — not boilerplate scaffolding — to learn naming (`PascalCase.tsx` vs. `kebab-case.tsx`), export style (default vs. named), props typing (inline literal vs. a separate `interface`/`type`), and the class-composition helper actually in use (`cn`, `clsx`, string concatenation).

**Match what you find. Do not impose a style of your own**, even one you consider objectively better. If no frontend stack is detected at all — no recognizable framework dependency, no styling system, nothing to read conventions from — stop and ask the user what to target rather than guessing.

## Step 3 — Analyze and propose a slicing plan (do not write files yet)

**Before proposing a plan for any non-trivial screen, read `references/slicing-patterns.md`.** It has a worked end-to-end example against a real measured screen, a catalogue of recurring Stitch screen patterns and how to cut each, granularity guidance for the hardest judgment call in this skill, and how to reconcile the downloaded HTML's inline `tailwind.config` palette with the project's own tokens. Do not improvise slicing heuristics from scratch when that reference exists.

Look for these seams in the downloaded markup:

- **Repeated sibling structures with identical (or near-identical) class signatures** — the strongest signal. A `<section>` containing nine `<article>` elements sharing the same class list and internal shape is one component plus a data array, not nine hand-copied blocks. `article`×9 inside one `section` is the canonical case worth training your eye on.
- **Semantic landmarks** (`header`, `nav`, `main`, `footer`, `aside`) — these map naturally to layout components (`SiteHeader`, `PrimaryNav`, `PageFooter`) that wrap page-level composition, distinct from the content components nested inside `main`.
- **Elements distinguished only by content** (same markup shape, different text/images) — these become props on a single component, not separate components.
- **Elements distinguished by class variations** (same shape, different Tailwind classes for state or emphasis — a "featured" card, a disabled button) — these become a variant prop (`variant="featured"`, `disabled`), not a second component that duplicates the first with small edits.

Present the plan as a table, one row per proposed component:

| Component | Source anchor | Instances | Proposed props | Target file |
|---|---|---|---|---|
| `ArticleCard` | `<section class="...">` → each `<article>` | 9 | `title`, `excerpt`, `imageUrl`, `href` | `components/ArticleCard.tsx` |
| `SiteHeader` | `<header>` | 1 | `logoHref`, `navItems` | `components/SiteHeader.tsx` |

"Source anchor" means enough of the markup (a tag, an id, a distinguishing class) that the user can locate it in the downloaded HTML without you re-pasting the whole block. **Then stop and ask for approval.** Do not create, write, or overwrite any component file before the user has responded to this table. If the user asks for changes to the plan, revise the table and ask again rather than writing a compromise you invented.

## Step 4 — Extract

Once the plan is approved, convert each planned component into the detected framework's syntax:

- **Keep Tailwind classes as-is** wherever the project itself uses Tailwind — the downloaded HTML's classes are already valid Tailwind utility classes; don't rewrite them into a different styling system unless the project has none.
- **Map the inline `tailwind.config` palette into the project's own token mechanism** rather than hardcoding hex values pulled straight from the downloaded HTML's Material-3 slot names (`surface-container-low`, `on-primary-fixed-variant`, etc.) into component files. See `references/slicing-patterns.md` for the reconciliation procedure.
- **Convert HTML attributes to the framework's conventions** — `class` → `className` (React), event-ready `data-*` attributes preserved as-is unless the framework has an idiomatic equivalent, `for` → `htmlFor`, and so on.
- **Lift literal text into props or a content object** per the approved plan — don't leave nine near-identical `<article>` blocks with only their text hardcoded differently after claiming to have "componentized" them.
- **Preserve semantic tags and accessibility attributes.** The downloaded HTML already carries meaningful ids (`header-search-input`, `hero-title`) and semantic elements — keep `<nav>`, `<article>`, `aria-*` attributes, and similar through the conversion rather than collapsing everything to `<div>`, which is the single most common way a slicing pass quietly degrades accessibility.

## Step 5 — Verify

Run whatever the project actually provides for typechecking, linting, and building — check `package.json` scripts and the presence of `tsconfig.json` for a raw fallback. Report honestly what passed and what did not, including partial results ("lint passes; no build script exists so the build step was skipped"). Do not invent a verification command the project lacks, and do not claim success you haven't actually observed in command output.

## Honesty rules

- **A Stitch screen is a static design, not a specification of behavior.** Its markup shows what a search input or a "Load more" button looks like, not what happens when a user interacts with it. Do not invent event handlers, routing, data fetching, or form submission the markup doesn't specify. Render interactive-looking elements as inert props or clearly flagged `// TODO` markers, and surface each such decision to the user rather than quietly filling it in.
- **Do not claim visual fidelity you haven't verified.** Extracting structure and classes into components is not the same as confirming the rendered result matches the Stitch screen pixel-for-pixel. Say what you translated and on what basis — a downloaded screenshot you actually looked at, or the markup alone — not "this matches exactly."
- **Never claim to have downloaded or read a file you have not.** If a `curl` failed, if a `downloadUrl` expired, if you skipped fetching the screenshot, say so plainly instead of describing the screen's content as if you'd seen it.
- If a class-variation case is genuinely ambiguous between "one component with a variant prop" and "two separate components," say so in the plan table rather than silently picking one — see the granularity guidance in `references/slicing-patterns.md`.
