# Skills reference

`stitch-skills` wraps the Google Stitch MCP server (`stitch`, tools exposed as `mcp__stitch__<tool>`) in eight skills plus one subagent, covering the path from a blank Stitch project to a themed screen, a component library, a Storybook, and application code. The Stitch MCP surface has sharp edges — bare ids versus prefixed resource names, screen *instance* ids that aren't screen ids, tool pairs that must be called back-to-back, multi-minute generations that must never be retried — and the skills encode those rules consistently. Skills activate on their own from natural-language requests; there are no slash commands to memorize. This is the deeper companion to [`README.md`](README.md), which covers installation and prerequisites.

## Quick reference

| Skill | Purpose | Example prompt |
|---|---|---|
| [`stitch-workflow`](skills/stitch-workflow/SKILL.md) | Entry point for multi-step or ambiguous Stitch work; sequences the other skills and carries ids across the session | "Design a fintech onboarding app with Stitch and take it all the way to React" |
| [`stitch-projects`](skills/stitch-projects/SKILL.md) | Finds, creates, inspects, deletes projects; the authoritative bare-vs-prefixed id reference | "Find my Acme project and show me its screens" |
| [`stitch-design-system`](skills/stitch-design-system/SKILL.md) | Creates/applies theme tokens (color, font, roundness) and exports or imports a DESIGN.md | "Give this project a clean, corporate look with our brand blue #2563EB" |
| [`stitch-screens`](skills/stitch-screens/SKILL.md) | Generates a brand-new screen from a text prompt | "Generate a mobile onboarding screen for a budgeting app" |
| [`stitch-iterate`](skills/stitch-iterate/SKILL.md) | Edits an existing screen in place, or generates alternative variants to compare | "Add a persistent bottom nav bar to the dashboard screen" |
| [`stitch-components`](skills/stitch-components/SKILL.md) | Downloads a screen's real HTML and slices it into a reviewable component plan | "Slice the dashboard screen into reusable React components" |
| [`stitch-storybook`](skills/stitch-storybook/SKILL.md) | Scaffolds or extends Storybook, writes per-component stories, composes page-level stories | "Set up Storybook for these components and build a page story for the dashboard" |
| [`stitch-to-code`](skills/stitch-to-code/SKILL.md) | Translates a screen and its design tokens into application code matching the host project's stack | "Turn the onboarding screen into a Next.js page that matches our stack" |

## The pipeline

```
stitch-projects ──> stitch-design-system ──> stitch-screens ──> stitch-iterate
                                                                     │
                                           ┌─────────────────────────┴─────────────────────────┐
                                           ▼                                                    ▼
                                   stitch-components                                    stitch-to-code
                                (screen HTML → components)                       (screen + tokens → app code)
                                           │
                                           ▼
                                   stitch-storybook
```

`stitch-workflow` sits above this: it decides which specialist skill a request maps to and carries `projectId`, design system `assetId`, and screen ids between them. The first four skills run roughly in the order shown — a project needs a design system before screens generate against it, then screens are edited or varied. Past that point the path forks: `stitch-components` and `stitch-to-code` are **alternative** routes to code, not sequential stages. The first slices a screen's markup into a reusable component library; the second translates a screen and its tokens directly into application code. A user may want either, both, or neither — `stitch-storybook` only makes sense once components exist, whether from `stitch-components` or written by hand.

## `stitch-workflow`

**What it does.** Sequences the other seven skills for multi-step or end-to-end work and tracks the ids (`projectId`, design system `assetId`, screen ids, screen instance ids, `deviceType`) that flow between them. It delegates to the matching skill, falling back to direct tool calls, per its own routing table, only when that skill isn't installed.

**When it fires.** "design an app with Stitch", "prototype this whole flow", "take this from design to code with Stitch", "set up a Stitch project from scratch", "I don't know where to start with Stitch", "take this Stitch design all the way to Storybook". Also Stitch MCP connection and authentication trouble. A single-step request goes straight to the matching specialist skill instead.

**What it needs.** Nothing upfront beyond the user's overall goal. It checks for a `stitch.local.md` settings file (`.codex/` or `.claude/`) for session defaults before asking the user to resolve a project or design system from scratch.

**Example.** "Design a fintech onboarding app with Stitch and get it all the way to React components."
1. `list_projects` to check for an existing match before creating one; `create_project` if nothing fits.
2. Delegate to `stitch-design-system`: `create_design_system` then `update_design_system`, establishing the theme before any screen exists.
3. Delegate to `stitch-screens`: `generate_screen_from_text`, passing the design system's `assetId` (`assets/{id}`).
4. Once screens are stable, hand off to `stitch-to-code` or `stitch-components` depending on what the user wants out of the design.

**Watch out for.** Ids tracked earlier in a session go stale after any generation, edit, or variant call — re-verify with `get_project`/`list_screens` rather than reusing a remembered value.

## `stitch-projects`

**What it does.** Owns project lifecycle — find, create, inspect, delete — and is the authority on which id format (bare or prefixed) each Stitch tool expects.

**When it fires.** "create a Stitch project", "list my Stitch projects", "find my existing project", "show the screens in this project", "get the screen instance id", "delete this Stitch project", or any question about which id format a tool expects.

**What it needs.** A title or description to search for, or nothing for a fresh listing. Deletion needs the user's explicit, fresh confirmation naming the specific project.

**Example.** "Find my Acme project and show me its screens."
1. `list_projects({ filter: "view=owned" })`, matching by title (try `"view=shared"` if it doesn't turn up).
2. `get_project({ name: "projects/{id}" })` for full detail including screen instance ids, if the next step needs them.
3. `list_screens({ projectId })` for a quick roster of bare screen ids otherwise.

**Watch out for.** `get_project`, `get_screen`, and `delete_project` are the exceptions that want the `projects/...` prefix; almost every other tool wants a bare id. Screen **instance** ids come only from `get_project` — a source screen id must never stand in for one.

## `stitch-design-system`

**What it does.** Configures color, typography, and shape tokens for a project — not screen content — and runs in both directions: authoring a system from structured tokens or a DESIGN.md, and exporting an existing project's resolved theme back to local files.

**When it fires.** Creating or updating a design system, setting brand colors, picking fonts, adjusting roundness, uploading or applying a DESIGN.md, or "export DESIGN.md from Stitch", "get the design tokens out of Stitch", "save the Stitch theme to a file", "match our brand", "look clean and corporate", "feel more playful".

**What it needs.** Either concrete brand values (hex color, named fonts, a style adjective — Path A) or a written design document (Path B). Applying a system to existing screens needs their screen instance ids from `get_project`.

**Example.** "Give this project a clean, corporate look with our brand blue #2563EB."
1. Map "clean & corporate" to the recipe's `headlineFont`/`bodyFont`/`colorVariant`/`roundness`, substituting the given hex.
2. `create_design_system({ projectId, designSystem: {...theme} })`.
3. `update_design_system({ name: "assets/{id}", projectId, designSystem: {...same theme} })` — immediately, no other calls between.

**Watch out for.** Both authoring paths are two-call sequences, and stopping after the first is the most common failure: `create_design_system` alone leaves an asset invisible and unapplied; `upload_design_md` alone leaves an orphaned document with zero theming applied.

References: `references/theme-enums.md` (before emitting any font, `colorVariant`, `roundness`, or `deviceType` value not already in the recipe table), `references/design-md.md` (before starting DESIGN.md work), `references/design-export.md` (before any export).

## `stitch-screens`

**What it does.** Generates a new screen from a text prompt via `generate_screen_from_text`. Changing a screen that already exists is `stitch-iterate`'s job.

**When it fires.** "generate a screen", "create a mockup", "make a new page design", "design a login page in Stitch", "build me a dashboard mockup".

**What it needs.** A resolved `projectId` (bare); a design system `assetId` passed as `designSystem: "assets/{id}"` (prefixed) — optional on the tool signature but treated as required in practice; a `deviceType` kept consistent for the project.

**Example.** "Generate a mobile onboarding screen for a budgeting app."
1. Resolve `projectId` and the design system `assetId`.
2. Write a full prompt per `references/prompting.md` rather than a one-liner.
3. `generate_screen_from_text({ projectId, prompt, designSystem: "assets/{id}", deviceType: "MOBILE" })`.
4. On a timeout, poll `get_screen` every 30 seconds, up to 10 attempts, instead of retrying.

**Watch out for.** The tool's own documentation says "DO NOT RETRY" — a slow or dropped-connection response isn't a failure, and re-issuing the call produces duplicate screens and burned quota. Poll `get_screen` instead.

Reference: `references/prompting.md` — open before writing any non-trivial prompt.

## `stitch-iterate`

**What it does.** Refines screens that already exist: `edit_screens` for a directed, known change applied in place, `generate_variants` for exploring undecided alternatives added alongside the original.

**When it fires.** "change the screen", "make it darker", "try another version", "show me variants", "tweak this screen", "give me a few options for this layout".

**What it needs.** `projectId` (bare), `selectedScreenIds` (bare, from `list_screens` or a prior generation), and a `prompt`. `generate_variants` also needs a `variantOptions` object (`variantCount`, `creativeRange`, `aspects`).

**Example.** "Add a persistent bottom nav bar with Home, Search, and Profile to the dashboard screen."
1. Confirm the target screen id if there's any ambiguity — `edit_screens` mutates in place with no preview.
2. `edit_screens({ projectId, selectedScreenIds: [...], prompt: "Add a persistent bottom navigation bar with three tabs..." })`.
3. On a timeout, poll `get_screen` as with generation.

**Watch out for.** A prompt written with one screen in mind, then broadcast across a heterogeneous `selectedScreenIds` list, misfires on screens it wasn't written for. `edit_screens` also has no preview step — if the original should stay intact, use `generate_variants` instead.

## `stitch-components`

**What it does.** Downloads a screen's real HTML (`get_screen` doesn't return markup inline) and slices it into components matched to the host project's own conventions. Its defining behavior is propose-then-write: it produces a slicing plan and stops for explicit approval before creating or overwriting any file.

**When it fires.** "slice this Stitch screen into components", "extract components from a Stitch design", "componentize the Stitch mockup", "turn the Stitch screen into a component library", "find the repeated blocks in this Stitch screen".

**What it needs.** An existing generated screen. Before proposing anything, it reads the host project's `package.json`, Tailwind config (or its absence, for v4's CSS-first `@theme`), `components.json`, `tsconfig.json`, and a few existing components to learn real conventions.

**Example.** "Slice the dashboard screen into reusable React components."
1. `list_screens` (if needed) then `get_screen({ name: "projects/{project}/screens/{screen}" })` for `htmlCode.downloadUrl`.
2. `curl -sL -o <file> "<downloadUrl>"` into a scratch directory — no auth needed.
3. Read `package.json`, Tailwind config, and `components.json` to detect the host stack.
4. Present a slicing plan as a table and stop for approval.
5. After approval, extract each component, mapping the inline palette onto the project's own tokens.

**Watch out for.** Never write a component file before the user has responded to the proposed plan — a wrong slicing decision compounds across everything built on it. The download URL is a signed, expiring link; re-call `get_screen` for a fresh one rather than reusing an old one.

Reference: `references/slicing-patterns.md` — open before proposing a plan for any non-trivial screen.

## `stitch-storybook`

**What it does.** Takes components — usually from `stitch-components`, or hand-written — to a working Storybook: scaffolding if absent, wiring the project's real design tokens and styling in, writing per-component stories, and composing page-level stories that reassemble components into the original screen's layout.

**When it fires.** Scaffolding Storybook, registering a component as a story, building a Storybook page from Stitch-derived components, or wiring backgrounds matching Stitch's theme. Also plain "set up Storybook" or "add a story for this component" in a project that already has Stitch-derived components, even without Stitch named explicitly.

**What it needs.** Components already existing in the project, and its detected framework/bundler/styling (React, Vue, Svelte, Angular, or plain web components — never assumed). If Storybook is already configured, its actual `.storybook/main.*`/`preview.*` files are read and extended, never re-initialized.

**Example.** "Set up Storybook for these components and build a page story for the dashboard."
1. Check `package.json`/`.storybook/` for an existing install; if none, confirm with the user, then run `npx storybook@latest init`.
2. Read what the initializer actually produced rather than assuming a config shape.
3. Wire the stylesheet, `namedColors`-derived backgrounds, and `deviceType`-mapped viewports into `.storybook/preview.*`.
4. Write a default story plus one per meaningful variant per component, using real screen content.
5. Compose a page-level story from the real components, using `screenshot.downloadUrl` as the visual reference.
6. Run the project's Storybook build script and report the actual result.

**Watch out for.** Re-running the initializer over an already-configured Storybook is destructive — it can overwrite addon lists and config the project depends on; extend the existing config instead. Never add Stitch's `https://cdn.tailwindcss.com` script to the real project — that's a Stitch preview convenience, not for a project with Tailwind actually installed.

Reference: `references/storybook-recipes.md` — read once the stack is known, for per-framework config recipes and a worked `ArticleCard`/`HomePage` example.

## `stitch-to-code`

**What it does.** Translates a finished Stitch screen plus its design system into working application code in the user's own project, matching that project's existing framework, styling system, and conventions rather than imposing a preferred stack.

**When it fires.** "turn this Stitch screen into code", "implement this Stitch design in my app", "build this Stitch screen as a Next.js page", "convert the Stitch design into shadcn components", "code up the screen we just generated in Stitch".

**What it needs.** An existing generated screen (or several), its design system's resolved tokens, and the host project's real stack — read from `package.json`, Tailwind config, `components.json`, `tsconfig.json`, and a few existing components, never assumed from what the user says the stack is.

**Example.** "Turn the onboarding screen into a Next.js page that matches our stack."
1. Detect the stack: React/Next.js, Tailwind version, shadcn/ui via `components.json`, TypeScript via `tsconfig.json`.
2. `list_screens` then `get_screen` for `htmlCode.downloadUrl` and `screenshot.downloadUrl`; download the HTML.
3. `list_design_systems` for resolved tokens (`namedColors`, `typography`, `spacing`, font families, `roundness`).
4. Establish those tokens in the project's own mechanism before writing any component.
5. Decompose the screen into components, keeping page composition in the route file and reusing existing primitives.
6. Run the project's lint/typecheck/build scripts and report the real result.

**Watch out for.** `roundness` selects a four-step border-radius scale (`DEFAULT`/`lg`/`xl`/`full`), not one pixel value — the enum name approximates the `lg` step only, and `rounded-full` is a true pill only under `ROUND_EIGHT`/`ROUND_TWELVE`; the screen's own inline `tailwind-config` script is authoritative, the measured table is only a fallback. A Stitch screen is a design, not a behavior spec — this skill won't invent business logic, API calls, or routes a static screen can't specify, and flags implied behavior instead of guessing at it.

Reference: `references/stack-recipes.md` — read once the stack is identified, for copy-ready Tailwind v4, shadcn CSS-variable, and `next/font/google` recipes.

## The `stitch-batch-generator` subagent

[`agents/stitch-batch-generator.md`](agents/stitch-batch-generator.md) generates several Stitch screens from a list of specs, strictly one at a time, against a shared `projectId`, `designSystem`, and `deviceType`. Each `generate_screen_from_text` call is long-running (a few minutes, no retries, poll-only recovery), so running a batch of five screens inline would stall the conversation on every one; the subagent absorbs that waiting and reports back once with a ledger — one row per requested screen, its `screenId` (or blank), and its status (`succeeded`, `failed`, or `unconfirmed`).

Dispatch it instead of generating screens one at a time whenever a request implies several related screens at once — "generate all five onboarding screens," "create the whole dashboard flow." A single screen doesn't need it; call `generate_screen_from_text` directly. It's invoked like any Claude Code subagent — the calling conversation dispatches it with the spec list and shared parameters — and has only `mcp__stitch__generate_screen_from_text`, `mcp__stitch__get_screen`, `mcp__stitch__list_screens`, and `Read`, so the dispatcher must hand it an already-resolved `projectId` and, if applicable, `designSystem`.

## Per-project settings

[`examples/stitch.local.md`](examples/stitch.local.md) is a template for per-project session defaults, copied into the consuming project as `.codex/stitch.local.md` (Codex) or `.claude/stitch.local.md` (Claude Code) and filled in there. Every skill and the batch-generator subagent read it when present, treating its values as defaults an explicit, contrary user instruction still overrides.

It carries four fields: `projectId` (bare), `designSystemAssetId` (bare), `deviceType` (`DEVICE_TYPE_UNSPECIFIED | MOBILE | DESKTOP | TABLET | AGNOSTIC`, kept consistent for the project's life), and `modelId` (`MODEL_ID_UNSPECIFIED | GEMINI_3_8_FLASH | GEMINI_3_5_FLASH_LITE`). `stitch-to-code` alone also reads an optional `targetStack` override for ambiguous or mid-migration `package.json` files.

The file holds no secrets — the Stitch MCP server authenticates through the `STITCH_API_KEY` environment variable, configured separately in the plugin's `.mcp.json`, never in `stitch.local.md`. If a field or the file is absent, the relevant skill falls back to resolving the value interactively.

## Install matrix

| Marketplace entry | What it gives you |
|---|---|
| `stitch-skills` | All 8 skills plus the `stitch-batch-generator` agent |
| `stitch-workflow` | Only `stitch-workflow` |
| `stitch-projects` | Only `stitch-projects` |
| `stitch-design-system` | Only `stitch-design-system` |
| `stitch-screens` | Only `stitch-screens` |
| `stitch-iterate` | Only `stitch-iterate` |
| `stitch-components` | Only `stitch-components` |
| `stitch-storybook` | Only `stitch-storybook` |
| `stitch-to-code` | Only `stitch-to-code` |

Every entry, including the eight single-skill ones, ships the same `.mcp.json`, so installing any one configures the `stitch` MCP server. The `stitch-batch-generator` agent is only declared explicitly on the `stitch-skills` entry, but agents under `agents/` in an installed plugin are auto-discovered, so it shows up even in a single-skill install — worth knowing since it isn't obvious from the marketplace listing. See [`README.md`](README.md) for the actual install commands.

## Appendix — verified API behaviour

The following four points were established by probing the live Stitch API on **2026-09-16**. Each corrects a natural but wrong assumption about the MCP surface.

1. **`get_screen` does not return markup.** It returns `title`, `deviceType`, `width`/`height` (as **strings**, not numbers), plus `htmlCode.downloadUrl` and `screenshot.downloadUrl`. The HTML must be downloaded separately — `curl -sL` against the URL returned HTTP 200 with no authentication, 44,631 bytes. The downloaded file is a standalone Tailwind document loading `https://cdn.tailwindcss.com`, with an inline `<script id="tailwind-config">` holding the resolved color palette.
2. **`designTheme` carries resolved tokens the write schema does not expose.** `namedColors` (47 snake_case→hex entries), `typography` (named levels such as `body-lg`, each `{fontFamily, fontSize, fontWeight, letterSpacing, lineHeight}`), `spacing` (named px tokens), and `bodyFontFamily`/`headlineFontFamily`/`labelFontFamily` as real CSS family names (observed `"Inter"`, `"Newsreader"`). Read these directly rather than deriving equivalents from the enum fields.
3. **`roundness` selects a four-step border-radius scale, not one value.** Measured from three real projects:

   | `roundness` | `DEFAULT` | `lg` | `xl` | `full` |
   |---|---|---|---|---|
   | `ROUND_FOUR` | 2px | 4px | 8px | 12px |
   | `ROUND_EIGHT` | 4px | 8px | 12px | 9999px |
   | `ROUND_TWELVE` | 8px | 16px | 24px | 9999px |

   Consequences: the enum name approximates the `lg` step, not a global radius; `ROUND_TWELVE` does **not** yield a 12px radius (its `lg` is 16px; 12px only appears as `ROUND_EIGHT`'s `xl`); and `rounded-full` is 12px under `ROUND_FOUR` rather than a true pill, only reaching `9999px` under `ROUND_EIGHT`/`ROUND_TWELVE`. `ROUND_FULL` wasn't present in the sample, so its scale is unknown.
4. **There is no DESIGN.md export tool.** Export means reading `designTheme.designMd` from `get_project`. It was populated on only 4 of 13 projects in the measured account, so its presence must be checked before claiming an export succeeded — never emit an empty file or fabricated prose in its place.

The complete set of 15 real Stitch MCP tools: `list_projects`, `create_project`, `get_project`, `delete_project`, `list_screens`, `get_screen`, `generate_screen_from_text`, `edit_screens`, `generate_variants`, `list_design_systems`, `create_design_system`, `update_design_system`, `apply_design_system`, `upload_design_md`, `create_design_system_from_design_md`.
