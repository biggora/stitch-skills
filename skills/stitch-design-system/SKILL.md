---
name: stitch-design-system
description: Configures visual theming and branding for Google Stitch projects through the stitch MCP server — design systems, design tokens, color palettes, typography, font pairings, headline and body fonts, corner radius, roundness, light and dark mode, spacing scales, and DESIGN.md-driven styling. Runs in both directions — authoring a design system into Stitch, and exporting an existing one out to local files. Applies when the user wants to create or update a Stitch design system, set brand colors, pick fonts, adjust roundness or corner radius, upload or apply a DESIGN.md or brand guidelines document, apply an existing theme to screens, or says "export DESIGN.md from Stitch", "get the design tokens out of Stitch", "download the Stitch design system", "save the Stitch theme to a file", "what colors does this Stitch project use", "match our brand", "look clean and corporate", or "feel more playful".
---

# Stitch Design System

Configure and apply visual themes for Google Stitch projects via the `mcp__stitch__*` tools. This skill governs color, typography, and shape tokens — not screen content or layout (see other `stitch-*` skills for that). It runs in both directions: the sections below through "Reference files" cover *authoring* a design system into Stitch (Path A or Path B); the "Exporting a design system out of Stitch" section covers the reverse — reading an existing project's theme back out to local files.

## Two paths to a design system

Every design system starts from one of two inputs. Pick the path based on what the user gives you.

**Path A — Structured tokens.** Use when the user states concrete brand values: a hex color, named fonts, "rounded corners," "dark mode," or a style adjective you can map to tokens (see the recipe table below). Call `create_design_system`, then immediately `update_design_system`.

**Path B — DESIGN.md.** Use when the user already has a written design document, brand guidelines file, or free-form prose describing their visual style at length (more than a few bullet points). Call `upload_design_md`, then immediately `create_design_system_from_design_md`. Full detail, including a worked example and the exact base64 steps for each OS, is in `references/design-md.md` — open it before running this path.

If the user gives you both a document and specific overrides ("use our brand doc, but make the primary color #FF0000"), prefer Path B for the base system, then follow up with `update_design_system` to layer the override on top.

## Mandatory call pairs — do not skip the second call

Both creation paths are two-call sequences. The Stitch tools' own documentation states this explicitly, and skipping the second call is the single most common failure mode:

1. **`create_design_system` → `update_design_system`, immediately, no other tool calls in between.** `create_design_system`'s own docs say: "Call `update_design_system` tool immediately after this tool to apply the design system to the project, and display the design system in the UI." If you stop after `create_design_system`, the asset exists but is **invisible and unapplied** — it will not show in the UI and will not affect any screen. There is no separate "apply" step for this path beyond the `update_design_system` call itself (that call both applies and syncs the UI).
2. **`upload_design_md` → `create_design_system_from_design_md`, immediately, no other tool calls in between.** The first call only registers the markdown and returns a screen instance; **no design system exists at all until the second call runs against that exact instance.** Stopping after `upload_design_md` leaves the project with an orphaned uploaded document and zero theming applied — there is nothing for the user to see, list, or apply.

Never treat either first call as a complete action. If you must stop mid-task, tell the user the design system is not yet applied rather than reporting the first call as done.

## Project-scoped vs. global design systems

`create_design_system` takes an optional `projectId` (bare id, no `projects/` prefix):

- **Pass `projectId`** when the design system belongs to one specific project — the normal case when a user is theming "this project."
- **Omit `projectId`** to create a **global** asset, not attached to any project. Choose this only when the user explicitly wants a reusable theme library across multiple projects (an agency starter kit, a multi-product brand system). Global systems still need `update_design_system` immediately after, and per the tool schema that call still takes a `projectId`; if you are updating a truly global asset, pass the project you are applying it to first.
- `list_design_systems` mirrors this: pass `projectId` to see that project's systems, omit it to list **global** systems only. The two scopes never overlap in one call — if you need both, call it twice.

Default to project-scoped unless the user says "reusable," "template," "across projects," or similar.

## Applying a design system to existing screens

`apply_design_system` pushes a design system onto specific screens that already exist. This is a separate operation from the create/update pair above — use it when the design system already exists (just created, or picked from `list_design_systems`) and the user wants it applied to screens beyond whatever `update_design_system` already touched.

Procedure, in order:

1. Call `get_project` for the target project. **Do not use `list_screens` for this** — it does not return screen instance ids, only source screen ids.
2. From `get_project`'s response, find each screen you want themed and extract **both** of its identifiers: the **screen instance id** (`id`) and the **source screen** resource name (`sourceScreen`, format `projects/{project}/screens/{screen}`).
3. Build `selectedScreenInstances` as an array of `{id, sourceScreen}` objects, one per screen.
4. Call `apply_design_system` with:
   - `projectId` — bare id.
   - `assetId` — bare id of the design system, **without** the `assets/` prefix.
   - `selectedScreenInstances` — the array from step 3.

Two traps, called out because they are easy to get backwards:

- **`id` is the screen *instance* id, explicitly documented as NOT the source screen id.** Passing a source screen id in the `id` field will not resolve to the right screen. Both values come only from `get_project`.
- **`assetId` (bare) vs. `name` (`assets/`-prefixed) is asymmetric across tools.** `update_design_system` takes `name` with the `assets/` prefix; `apply_design_system` takes `assetId` without it. Do not copy one value into the other tool's field unmodified — strip or add the prefix as needed.

## Exporting a design system out of Stitch

There is no export tool. None of the 15 `mcp__stitch__*` tools downloads a DESIGN.md or a token file — `upload_design_md` is write-only, one direction only, and has no counterpart that reads a document back out. Export means reading the `designTheme` field off a project via `get_project` and writing what you find to local files yourself. Treat "export," "download," or "save the theme" requests as this read-plus-write procedure, not as a search for a missing tool.

**Before performing an export, open `references/design-export.md`.** It has the full `designTheme` field inventory (which fields are writable vs. resolved-only), a worked example producing a DESIGN.md, a CSS token file, and a Tailwind fragment from real values, the `namedColors` casing-normalization rule, and a synthesis template for when no DESIGN.md exists.

Procedure, in order:

1. **Resolve the project.** If you don't already have the project id, call `list_projects` (optionally with `filter: view=owned` — the default — or `view=shared`) and match it by title. Otherwise call `get_project` directly; its `name` parameter takes the **prefixed** form `projects/{id}`, unlike the bare `projectId` used elsewhere in this skill.
2. **Read `designTheme`** off the `get_project` response.
3. **Write the requested artefacts to disk** — see the table below for what each one is built from.

| Artefact | Built from | Notes |
|---|---|---|
| `DESIGN.md` | `designTheme.designMd` | Only when present — see the missing-`designMd` path below. |
| Design-tokens file (CSS custom properties, Tailwind fragment, JSON, etc.) | `namedColors` + `typography` + `spacing` + the `*FontFamily` fields + `roundness` | Your synthesized format; `references/design-export.md` has worked examples in three formats. |
| Raw `designTheme` JSON | `designTheme` itself | A verbatim dump for reference or debugging — no transformation. |

**The missing-`designMd` path.** `designTheme.designMd` is not always populated — in the account this skill was verified against, only 4 of 13 projects had one. Check for it explicitly before claiming an export succeeded. If it is absent:

- Say so plainly — do not report a DESIGN.md export as done when there was nothing to export.
- Offer to synthesize one instead, built from the resolved tokens (`namedColors`, `typography`, `spacing`, the font family fields, `roundness`) rather than the prose original. `references/design-export.md` has the synthesis template — it also says what to leave out rather than invent (brand narrative, component tone, "what to avoid" — none of that is recoverable from tokens).
- Never emit an empty file or fabricate prose to fill the gap.

**Output location.** Default to writing artefacts at the project root, or to a path the user names. Since a hand-maintained `DESIGN.md` is common, confirm with the user before overwriting an existing one rather than silently replacing it.

**The round trip.** An exported (or synthesized) `DESIGN.md` can be edited locally and pushed back into Stitch through the `upload_design_md` → `create_design_system_from_design_md` pipeline this skill already documents in full in `references/design-md.md` — do not re-derive those steps here.

## Choosing theme values from a user's brief

When the user describes intent rather than literal tokens, map it through this procedure:

1. Identify the closest style adjective(s) in the recipe table below.
2. Take that recipe's `headlineFont`, `bodyFont`, `colorVariant`, and `roundness` as your starting point.
3. Replace `customColor` with the user's actual brand hex if they gave one; otherwise keep the recipe's sample color.
4. If the user names a font that isn't in the recipe, swap it in — but first confirm it's a valid enum value in `references/theme-enums.md` (see next section).
5. If nothing in the brief indicates dark mode, default `colorMode` to `LIGHT`.

### Theme recipes

Each pairs a distinctive headline face with a body face chosen for compatibility at reading sizes — not two headline faces fighting each other.

| Style intent | `headlineFont` | `bodyFont` | `colorVariant` | `roundness` | Sample `customColor` |
|---|---|---|---|---|---|
| Clean & corporate | `PLUS_JAKARTA_SANS` | `INTER` | `NEUTRAL` | `ROUND_EIGHT` | `#2563EB` |
| Playful & friendly | `SPACE_GROTESK` | `WORK_SANS` | `VIBRANT` | `ROUND_FULL` | `#F97316` |
| Editorial / magazine | `PLAYFAIR_DISPLAY` | `SOURCE_SERIF_4` | `TONAL_SPOT` | `ROUND_FOUR` | `#7C2D12` |
| Brutalist / raw | `ANTON` | `IBM_PLEX_SANS` | `MONOCHROME` | `ROUND_FOUR` | `#000000` |
| High contrast & accessible | `ATKINSON_HYPERLEGIBLE_NEXT` | `ATKINSON_HYPERLEGIBLE_NEXT` | `NEUTRAL` | `ROUND_EIGHT` | `#1D4ED8` |
| Tech / SaaS dashboard | `GEIST` | `INTER` | `TONAL_SPOT` | `ROUND_TWELVE` | `#6366F1` |
| Warm editorial / lifestyle | `NEWSREADER` | `NUNITO_SANS` | `EXPRESSIVE` | `ROUND_TWELVE` | `#DB2777` |

Notes on the choices: Plus Jakarta Sans is a geometric grotesque restrained enough to sit above Inter's neutral body without clashing. Space Grotesk's quirky proportions read as energetic at headline size while Work Sans stays warm and legible smaller. Playfair Display's high-contrast strokes need a plain text serif underneath, not another display face — Source Serif 4 is built for exactly that role. Anton is a headline-only face by construction (see `references/theme-enums.md`); pairing it with a neutral technical sans like IBM Plex Sans keeps body copy readable. Atkinson Hyperlegible Next is purpose-built for legibility, so it's the one recipe using the same font for both roles. Geist plus Inter is the de facto modern SaaS pairing. Newsreader's literary warmth needs a soft, rounded-terminal sans underneath — Nunito Sans — to avoid feeling stuffy.

For brutalist requests wanting fully square corners, note that `ROUND_FOUR` is the smallest concrete corner value available (besides the unused `ROUND_TWO`); `ROUNDNESS_UNSPECIFIED` is the alternative if any visible rounding is unacceptable, since it leaves the value unset rather than forcing a small radius.

`roundness` selects a four-step border-radius scale, not one fixed pixel value — the enum name approximates that scale's `lg` step, not a global radius applied everywhere. See `references/theme-enums.md` for the measured scale.

## Never invent enum values

`headlineFont`, `bodyFont`, `labelFont`, `colorVariant`, `roundness`, and `deviceType` are closed enums. **Before emitting any value for these fields that is not already in this file's recipe table, open `references/theme-enums.md` and confirm the exact string.** Do not guess a plausible-looking font name (e.g. do not emit `"ROBOTO"` or `"HELVETICA"` — they are not in the enum) and do not paraphrase a value's casing.

Avoid the three deprecated fonts — `SOURCE_SERIF_FOUR`, `METROPOLIS`, `SOURCE_SANS_THREE` — even if the user names them by their plain-English family name; substitute the non-deprecated sibling where one exists (`SOURCE_SERIF_4` for Source Serif, `SOURCE_SANS_3` for Source Sans). `references/theme-enums.md` also flags fonts unsuitable for body text (all-caps display faces, ultra-thin faces, monospace faces used decoratively) — check it before assigning any font to `bodyFont` specifically, since a headline-only face there will hurt readability.

## Worked example: structured tokens (Path A)

A "clean & corporate" system for a project, using the recipe above with the user's real brand blue:

```json
// Call 1
{
  "tool": "mcp__stitch__create_design_system",
  "input": {
    "projectId": "a1b2c3d4",
    "designSystem": {
      "displayName": "Acme Corporate Theme",
      "theme": {
        "colorMode": "LIGHT",
        "headlineFont": "PLUS_JAKARTA_SANS",
        "bodyFont": "INTER",
        "roundness": "ROUND_EIGHT",
        "customColor": "#2563EB",
        "colorVariant": "NEUTRAL"
      }
    }
  }
}

// Response includes something like: { "name": "assets/ds_9f8e7d6c" }

// Call 2 — immediately after, same theme, now applied
{
  "tool": "mcp__stitch__update_design_system",
  "input": {
    "name": "assets/ds_9f8e7d6c",
    "projectId": "a1b2c3d4",
    "designSystem": {
      "displayName": "Acme Corporate Theme",
      "theme": {
        "colorMode": "LIGHT",
        "headlineFont": "PLUS_JAKARTA_SANS",
        "bodyFont": "INTER",
        "roundness": "ROUND_EIGHT",
        "customColor": "#2563EB",
        "colorVariant": "NEUTRAL"
      }
    }
  }
}
```

Note `name` in call 2 carries the `assets/` prefix returned by call 1 — do not strip it here (contrast with `apply_design_system`'s bare `assetId`, described above).

## Reference files — when to open them

- **`references/theme-enums.md`** — open before emitting any font, `colorVariant`, `roundness`, or `deviceType` value not already copied verbatim from the recipe table in this file. It is the exhaustive, authoritative enum list. It also documents the shape of the non-enum `spacing` and `typography` maps (free-form string keys and CSS-value strings, not closed enums) — check it there too before constructing either.
- **`references/design-md.md`** — open before starting Path B (any DESIGN.md work): base64 encoding commands per OS, the full two-call sequence, how to obtain `selectedScreenInstance`, an example DESIGN.md, and when to use a DESIGN.md versus the `theme.designMd` field on a structured system.
- **`references/design-export.md`** — open before performing any export (the reverse direction, above): the full `designTheme` field inventory with writable-vs-resolved status, a worked example producing a DESIGN.md, a CSS token file, and a Tailwind fragment from one real theme, the `namedColors` snake_case-to-kebab-case normalization rule, and the synthesis template for a missing `designMd`.
