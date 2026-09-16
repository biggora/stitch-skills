<!--
  TEMPLATE — this is documentation, not live configuration.

  Copy this file to `.claude/stitch.local.md` inside the project that consumes
  the Stitch skills, then fill in the values for that project. The `.claude/*.local.md`
  glob is git-ignored by convention (see this repository's own `.gitignore`), so a
  copy placed there is never committed and is local to your machine/checkout.

  This file contains NO secrets. The Stitch MCP server authenticates with the
  STITCH_API_KEY environment variable (see .mcp.json's "X-Goog-Api-Key" header) —
  never put an API key, token, or credential in this file. Everything below is
  non-secret project metadata: ids, enum choices, and a stack override.

  Every skill in this plugin (stitch-projects, stitch-screens, stitch-design-system,
  stitch-iterate, stitch-to-code, and the stitch-batch-generator agent) should read
  this file when present, to skip resolving these values interactively each time.
  If the file is absent, or a field inside it is absent, the skill falls back to
  asking the user or resolving the value through the relevant Stitch MCP tool
  (list_projects, list_design_systems, etc.) as documented in that skill's SKILL.md.
-->

# Stitch project settings

```yaml
# projectId — the Stitch project this checkout works against.
#
# Bare id, WITHOUT the "projects/" prefix (e.g. "4044680601076201931", not
# "projects/4044680601076201931") — most Stitch MCP tools that take a project
# id want it bare; a few (get_project) want it prefixed, and the skills add
# that prefix themselves when needed.
#
# Where to find it: the id segment of a Stitch project URL, or the "id" field
# returned by the stitch `list_projects` tool.
#
# If omitted: every skill falls back to `list_projects` and asks you to pick
# (or create) a project at the start of the task.
projectId: ""

# designSystemAssetId — the design system to apply to every screen generated
# or edited in this project, so styling stays consistent across the whole set.
#
# Bare id, WITHOUT the "assets/" prefix. Skills that pass it to a Stitch tool
# (e.g. generate_screen_from_text's `designSystem` parameter) add the
# "assets/" prefix themselves — store it bare here.
#
# Where to find it: the "id" field returned by the stitch `list_design_systems`
# tool for this project, or `get_project` if the project already has one applied.
#
# If omitted: skills fall back to `list_design_systems` and ask which one to
# use (or offer to create one) before generating or editing screens. Screens
# generated with no design system at all drift stylistically from one another —
# see stitch-screens/SKILL.md.
designSystemAssetId: ""

# deviceType — the target form factor for every screen generated in this
# project. One of: DEVICE_TYPE_UNSPECIFIED | MOBILE | DESKTOP | TABLET | AGNOSTIC
#
# Keep this consistent for the life of the project — mixing MOBILE and DESKTOP
# screens in one project produces a visually incoherent set.
#
# If omitted: skills ask once per project (or infer from context, e.g. "an
# app screen" leans MOBILE, "an admin panel" leans DESKTOP) rather than
# leaving it unspecified by default.
deviceType: "DEVICE_TYPE_UNSPECIFIED"

# modelId — which Stitch generation model to use for
# generate_screen_from_text. One of:
# MODEL_ID_UNSPECIFIED | GEMINI_3_8_FLASH | GEMINI_3_5_FLASH_LITE
#
# GEMINI_3_5_FLASH_LITE is the smaller/faster option; GEMINI_3_8_FLASH is the
# stronger one. There's no published quality benchmark distinguishing them for
# screen generation specifically.
#
# If omitted: skills omit `modelId` on the tool call and let Stitch choose its
# own default, unless you ask for faster generation or explicitly name a model.
modelId: "MODEL_ID_UNSPECIFIED"

# targetStack — overrides stitch-to-code's Step 1 stack auto-detection.
#
# Free-form but should name a stack stitch-to-code's references/stack-recipes.md
# actually has a recipe for, e.g.: "Next.js App Router + Tailwind v4 + shadcn/ui",
# "Vue 3 + Nuxt + Tailwind v3", "SvelteKit + plain CSS".
#
# Use this when the project's package.json is ambiguous or mid-migration (e.g.
# both `tailwind.config.ts` and a v4 `@theme` block exist during an upgrade) and
# you want to pin which one stitch-to-code targets, instead of relying on
# detection to guess correctly every time.
#
# If omitted: stitch-to-code always runs its own stack detection (reading
# package.json, config files, and a few existing components) rather than
# assuming a stack from a stale note left in this file.
targetStack: ""
```
