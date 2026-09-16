---
name: stitch-workflow
description: Use for multi-step or end-to-end Google Stitch work, or when it is unclear which Stitch step a request needs. Applies when the user says things like "design an app with Stitch", "prototype this whole flow", "take this from design to code with Stitch", "set up a Stitch project from scratch", "what order should I do Stitch things in", "I don't know where to start with Stitch", "take this Stitch design all the way to Storybook", "get everything out of this Stitch project", or describes a UI job spanning project setup, design system, screens, component extraction, Storybook, and code handoff. Also covers Stitch MCP connection and authentication trouble. Sequences the specialist skills (stitch-projects, stitch-design-system, stitch-screens, stitch-iterate, stitch-components, stitch-storybook, stitch-to-code) and carries project, design system, and screen ids across the session. For a single step — one screen, one edit, one project lookup, one theme change — prefer the matching specialist skill.
---

# Stitch Workflow

This is the entry point for multi-step Google Stitch work (MCP server `stitch`, tools exposed as `mcp__stitch__<tool>`). It does not do the design work itself — it sequences the specialist skills (`stitch-projects`, `stitch-design-system`, `stitch-screens`, `stitch-iterate`, `stitch-components`, `stitch-storybook`, `stitch-to-code`) and tracks the identifiers that flow between them. When a request maps cleanly onto one step, hand off to that specialist skill and let it own the call; use the fallbacks in the routing table below only when the specialist is not installed.

## Canonical end-to-end sequence

1. **Resolve or create a project.** Call `list_projects` before `create_project` to avoid duplicating an existing project. Delegate to `stitch-projects`.
2. **Establish a design system before generating any screens.** Delegate to `stitch-design-system` (create, upload a design.md, or derive one from an existing screen instance).
3. **Generate screens against that design system.** Delegate to `stitch-screens`, passing the design system's `assetId` into `generate_screen_from_text`'s `designSystem` parameter.
4. **Iterate.** Refine screens with `edit_screens` or explore alternatives with `generate_variants`. Delegate to `stitch-iterate`.
5. **Route out of Stitch.** Once screens and the design system are stable, there is no single next step — what the user wants determines which of these to do, and they are not strictly sequential: export design tokens and/or a DESIGN.md from the design system (delegate to `stitch-design-system`), slice a generated screen into reusable framework components (delegate to `stitch-components`), scaffold or extend Storybook with those components and compose page stories (delegate to `stitch-storybook`), and/or hand off to `stitch-to-code` for full application code. `stitch-components` and `stitch-to-code` are alternative routes to code, not sequential steps: `stitch-components` produces a reusable component library, `stitch-to-code` produces application code, and a user may want either, both, or neither.

Do not reorder steps 2 and 3. Skip step 2 only if the user explicitly says the screen is disposable/throwaway and consistency does not matter — otherwise call it out and default to establishing a design system first.

### Exporting out of Stitch

The resolved design tokens (`namedColors`, `typography`, `spacing`, `bodyFontFamily`/`headlineFontFamily`/`labelFontFamily`) and the full DESIGN.md text (`designTheme.designMd`) both come back inside `designTheme` from `get_project` (`name: "projects/{id}"`). There is no dedicated export tool: exporting a DESIGN.md means reading `designTheme.designMd` from that response and writing it to a file yourself. `designMd` is not always present, so check that the field actually came back before telling the user an export succeeded. Delegate the detail work here to `stitch-design-system`; without it installed, the `get_project` call above is the fallback.

A minimal happy-path call sequence looks like this (ids are illustrative placeholders, not real values):

1. `list_projects` → no match → `create_project({ title: "Acme Checkout Redesign" })` → returns a project with bare id `abc123`.
2. `create_design_system({ projectId: "abc123", designSystem: {...} })` → returns asset `assets/ds456`.
3. `generate_screen_from_text({ projectId: "abc123", prompt: "...", designSystem: "assets/ds456", deviceType: "MOBILE" })` → long-running; poll if needed.
4. `get_project({ name: "projects/abc123" })` → confirm the new screen instance exists and capture its `{id, sourceScreen}`.
5. `edit_screens({ projectId: "abc123", selectedScreenIds: [...], prompt: "..." })` to refine, or `generate_variants` to branch.
6. Hand the finished screen ids and design system asset off to `stitch-to-code`.

### Why the design system comes first

`generate_screen_from_text` accepts an optional `designSystem` parameter, and the tool's own documentation states a design system "should always be configured for design consistency." Generating screens without one and fixing it later is strictly more expensive: it requires a separate `apply_design_system` call afterward, and that call needs `selectedScreenInstances` — an array of `{id, sourceScreen}` pairs that can only be obtained by calling `get_project` again after the screens exist. In other words, retrofitting costs an extra round trip through `get_project` plus one `apply_design_system` pass over every already-generated screen, on top of the original generation cost. Configuring the design system up front avoids all of that.

## State you must carry

Track these values for the duration of the session. All of them are plain strings/objects returned by tool calls — do not invent or guess a value that hasn't come back from a tool.

- **`projectId`** — bare id, no `projects/` prefix. Feeds `list_screens`, `generate_screen_from_text`, `edit_screens`, `generate_variants`, `apply_design_system`, `upload_design_md`, `list_design_systems`, `create_design_system`.
- **Design system `assetId`** — bare id, no `assets/` prefix, as required by `apply_design_system`. (Note: `update_design_system` instead wants this same asset as a prefixed `name`, `assets/{asset_id}` — see the identifier table in `stitch-projects` (if installed) for the full breakdown.)
- **Screen ids** — bare ids, no `screens/` prefix, used in `selectedScreenIds` arrays for `edit_screens` and `generate_variants`.
- **Screen instance ids** — `{id, sourceScreen}` pairs, obtainable only from `get_project`. Required by `apply_design_system` (`selectedScreenInstances`) and `create_design_system_from_design_md` (`selectedScreenInstance`). A screen instance id is **not** the same as a source screen id — never substitute one for the other.
- **Chosen `deviceType`** — pick once per project (`MOBILE`, `DESKTOP`, `TABLET`, or `AGNOSTIC`) and reuse it across every `generate_screen_from_text` call in that project so screens stay visually consistent. `DEVICE_TYPE_UNSPECIFIED` is the "let Stitch decide" default; avoid it once a project has a settled device target.

**Do not trust memory of these ids across a long session.** Screens and instances can change between turns (new screens generated, variants created, edits applied). Before any call that needs a screen id, screen instance id, or the current screen list, re-read the current state with `get_project` (for instances) or `list_screens` (for the screen roster) rather than reusing values you tracked earlier in the conversation.

## Session config

Before starting work, check for `stitch.local.md` in the consuming project's host settings directory: `.codex/` for Codex, `.claude/` for Claude Code. Resolve paths from the project root supplied by the host, or the current working directory; do not require `CLAUDE_PROJECT_DIR`. In Codex, fall back to `.claude/stitch.local.md` if `.codex/stitch.local.md` is absent, so existing project settings remain usable. If both exist, use only the host's file.

- **If it exists**, read it and use its contents as session defaults: `projectId`, `designSystemAssetId`, `deviceType`, `modelId`. Treat these as defaults, not overrides — if the user names a different project or device type explicitly, honor what they said instead.
- **If it does not exist**, do not create it silently. Once a project and design system are established in this session, offer to write one (with the resolved `projectId`, `designSystemAssetId`, `deviceType`, and, if chosen, `modelId`) so future sessions can skip re-resolving them. Only write it if the user agrees.

## Routing table

Each specialist skill may or may not be installed, since they are distributed individually. If a listed skill isn't available, use the fallback tool calls in the right-hand column so the workflow still completes without it.

| User intent | Skill to use | Fallback tool calls if that skill is not installed |
|---|---|---|
| Find, create, inspect, or delete a project; resolve any project/screen/instance id | `stitch-projects` | `list_projects` (optional `filter: "view=owned"` or `"view=shared"`) to find an existing project before `create_project` (optional `title`); `get_project` (`name: "projects/{id}"`) for full detail including screen instances; `delete_project` only after explicit user confirmation |
| Build, update, or apply a design system | `stitch-design-system` | `list_design_systems` (optional `projectId`, omit for global systems); `create_design_system` (`designSystem`, optional `projectId`); `update_design_system` (`name: "assets/{id}"`, `projectId`, `designSystem`); `apply_design_system` (`projectId`, bare `assetId`, `selectedScreenInstances` from `get_project`); `upload_design_md` (`projectId`, `designMdBase64`); `create_design_system_from_design_md` (`projectId`, `selectedScreenInstance`) |
| Generate new screens from a text description | `stitch-screens` | `generate_screen_from_text` (`projectId`, `prompt`, optional `designSystem`, `deviceType`, `modelId`); this is long-running — see Cost and Patience below |
| Refine an existing screen or explore alternatives | `stitch-iterate` | `edit_screens` (`projectId`, `selectedScreenIds`, `prompt`); `generate_variants` (`projectId`, `selectedScreenIds`, `prompt`, `variantOptions`); both are long-running |
| Turn finished screens/design tokens into real application code | `stitch-to-code` | No direct Stitch MCP tool performs this — it is genuinely outside the API reference available here. Without that skill installed, the only fallback is to inspect the finished work with `get_screen`/`get_project` and translate it into code by hand against the host project's stack. |
| Slice a generated screen's HTML into reusable framework components | `stitch-components` | `get_screen` (`name: "projects/{project}/screens/{screen}"`, both segments prefixed) does not return markup itself — read `htmlCode.downloadUrl` from the response, download it with `curl -sL {downloadUrl}` (no auth required), then slice the resulting standalone-Tailwind document (it loads `https://cdn.tailwindcss.com` and carries a `tailwind-config` script tag with the resolved palette) into components by hand |
| Set up Storybook, register components as stories, or compose a page story from a Stitch screen | `stitch-storybook` | No Stitch MCP tool is involved — Storybook itself is scaffolded with `npx storybook@latest init`; wire stories up by hand against whatever components already exist |

## Cost and patience

Screen generation, edits, and variants (`generate_screen_from_text`, `edit_screens`, `generate_variants`) each take minutes and consume quota. Do not fire off a string of small, single-tweak generations hoping to converge — write one well-specified prompt that captures the full intent (layout, content, states, copy, device target) and generate once. Asking the user a clarifying question before generating is cheaper than generating twice.

If a call appears to hang or the connection drops, do not retry it: retrying a long-running generation is explicitly forbidden by the underlying tools, and a second concurrent generation against the same screen can waste quota or produce conflicting results. A dropped connection does not mean the generation failed — poll `get_screen` (or `list_screens`) every 30 seconds, up to 10 attempts, to check whether it actually completed before taking any further action or telling the user it failed.

## Common mistakes to avoid

- Generating screens before a design system exists "to save time," then paying for it with an extra `apply_design_system` pass plus a `get_project` round trip to collect instance ids.
- Reusing a `projectId`, screen id, or screen instance id from earlier in the conversation without re-verifying it against a fresh `get_project`/`list_screens` call, especially after any generation, edit, or variant step that could have changed the project's screen list.
- Passing a source screen id where `apply_design_system` or `create_design_system_from_design_md` expects a screen instance id (`{id, sourceScreen}`) — see `stitch-projects` (if installed) for the full identifier reference.
- Retrying a `generate_screen_from_text`, `edit_screens`, or `generate_variants` call after a timeout or dropped connection instead of polling `get_screen` to check whether it already succeeded.
- Silently creating the host's `stitch.local.md` without asking, or ignoring it when it already exists.
