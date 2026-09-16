---
name: stitch-projects
description: Use when the user wants to find, list, create, inspect, or delete a Google Stitch project, or needs to resolve a Stitch identifier — a project id, screen id, screen instance id, or design system asset id — before calling another Stitch MCP tool. Applies when the user says things like "create a Stitch project", "list my Stitch projects", "find my existing project", "what are my project ids", "show the screens in this project", "show me the screen we generated", "get the screen instance id", "delete this Stitch project", "list shared projects", or asks which id format a Stitch tool expects.
---

# Stitch Projects

Owns project lifecycle (find, create, inspect, delete) and identifier resolution for the Google Stitch MCP server (`stitch`, tools exposed as `mcp__stitch__<tool>`). This skill is the authority on which id format — a bare id or a prefixed resource name — each Stitch tool expects. Consult it before calling any Stitch tool if there's doubt about an id's shape.

## Resolving a project

Prefer finding an existing project over creating a duplicate:

1. Call `list_projects` first. It takes an optional `filter` of `"view=owned"` (default) or `"view=shared"`. Search the results for a project matching the user's description (by title) before creating anything.
2. Only call `create_project` if no existing project fits. Pass a meaningful `title` — a string the user would recognize later in a `list_projects` call, not a generic placeholder like "New Project".
3. Once resolved, the project's bare id (the part after `projects/`) is what almost every other Stitch tool call needs. Keep it — see the identifier table below for exactly which form each tool wants.

## Identifier reference table

This is the exhaustive, tool-by-tool answer to "prefixed resource name or bare id?" Get this wrong and the call fails or silently targets the wrong resource — verify against this table rather than guessing from a similar-looking tool.

| Tool | Parameter | Id form required | Notes |
|---|---|---|---|
| `list_projects` | — | n/a | Optional `filter`: `"view=owned"` or `"view=shared"` |
| `create_project` | — | n/a | Optional `title` string only |
| `get_project` | `name` | **Prefixed**: `projects/{id}` | Only project-lookup tool that wants the prefix |
| `delete_project` | `name` | **Prefixed**: `projects/{id}` | Irreversible — see Deletion below |
| `list_screens` | `projectId` | **Bare**: `{id}` | No `projects/` prefix |
| `get_screen` | `name` | **Prefixed**: `projects/{project}/screens/{screen}` | Both segments prefixed |
| `generate_screen_from_text` | `projectId` | **Bare**: `{id}` | Optional `designSystem` param is `assets/{id}` (prefixed) |
| `edit_screens` | `projectId` | **Bare**: `{id}` | `selectedScreenIds` array is also bare, no `screens/` prefix |
| `generate_variants` | `projectId` | **Bare**: `{id}` | `selectedScreenIds` array is also bare |
| `list_design_systems` | `projectId` | **Bare**: `{id}`, optional | Omit entirely to list global design systems |
| `create_design_system` | `projectId` | **Bare**: `{id}`, optional | Omit to create a global asset instead of a project-scoped one |
| `update_design_system` | `name` | **Prefixed**: `assets/{asset_id}` | Also takes `projectId` (bare) and `designSystem` |
| `apply_design_system` | `assetId` | **Bare**: `{id}`, **no** `assets/` prefix | Contrast with `update_design_system`'s `name`, which is prefixed — same underlying asset, different parameter name and different id shape |
| `apply_design_system` | `selectedScreenInstances` | Array of `{id, sourceScreen}` objects | Screen **instance** ids, not screen ids — see next section |
| `upload_design_md` | `projectId` | **Bare**: `{id}` | Plus `designMdBase64` |
| `create_design_system_from_design_md` | `projectId` | **Bare**: `{id}` | Plus `selectedScreenInstance` (single `{id, sourceScreen}` object, not an array) |

The single most error-prone pair: `apply_design_system`'s `assetId` (bare) versus `update_design_system`'s `name` (prefixed `assets/{id}`). Both refer to the same design system asset — check which tool you're calling before formatting the id.

## Screen instance ids vs source screen ids

`apply_design_system` (`selectedScreenInstances`) and `create_design_system_from_design_md` (`selectedScreenInstance`) do not take screen ids. They take **screen instance** ids — a distinct identifier for a screen's placement inside a project, paired with its `sourceScreen`.

- The only way to obtain screen instance ids is `get_project`. Its response includes the project's screen instances, each with an `id` (the instance id) and a `sourceScreen` (the id of the screen it was generated from).
- `list_screens` returns the project's screens, but **not** instance ids — it cannot substitute for `get_project` when an instance id is required.
- **Never pass a source screen id where an instance id is expected.** They can look similar but are not interchangeable; passing the wrong one will target the wrong resource or fail. If a call needs `selectedScreenInstances` or `selectedScreenInstance`, call `get_project` first (or re-call it if the screen list may have changed since the last read) and pull the `{id, sourceScreen}` pair directly from its response — do not construct one from a screen id you already have lying around.

## Inspecting a project: which tool to use

- **`get_project`** (`name: "projects/{id}"`) — the only source of screen **instance** ids. Returns project-level info plus the list of screen instances (each with `id` and `sourceScreen`). Use this whenever the next step needs `selectedScreenInstances`/`selectedScreenInstance`, or when you need a fresh, authoritative picture of the project's current state.
- **`list_screens`** (`projectId`, bare) — a roster of the screens in a project. Use this for a quick listing or to get bare screen ids for `selectedScreenIds` in `edit_screens`/`generate_variants`. Cannot supply instance ids.
- **`get_screen`** (`name: "projects/{project}/screens/{screen}"`) — full detail on one specific screen. Use this to inspect a single screen's content, or to poll for completion after a long-running generation call (`generate_screen_from_text`, `edit_screens`, `generate_variants`) that timed out or dropped its connection — poll every 30 seconds, up to 10 attempts, rather than retrying the generation call itself.

In short: need instance ids → `get_project`. Need a screen roster or bare screen ids → `list_screens`. Need one screen's full content → `get_screen`.

## Deletion

`delete_project` (`name: "projects/{id}"`) is **irreversible**. The tool's own documentation requires confirming with the user before calling it, and this skill enforces the same rule:

- Always ask the user to explicitly confirm (a clear "yes"/"no") immediately before calling `delete_project`, even if they asked for the deletion earlier in the conversation.
- Never infer consent from an earlier, more general instruction (e.g., "clean up my test projects" mentioned several turns ago does not authorize deleting a specific project now without re-confirming which one and that it's really wanted).
- State plainly what will be deleted (the project's title and id) when asking for confirmation, so the user is confirming the right target.

## `view=owned` vs `view=shared`

`list_projects`' `filter` parameter controls which projects come back:

- `"view=owned"` (the default) — projects the user owns.
- `"view=shared"` — projects shared with the user by someone else.

If the user is looking for a project and it doesn't turn up under the default owned view, try `"view=shared"` before concluding the project doesn't exist or offering to create a new one.

## Worked example: resolving ids for a design-system pass

A concrete illustration of the id-shape rules above (ids are illustrative placeholders):

1. `list_projects({ filter: "view=owned" })` → find the target project, bare id `abc123`.
2. `get_project({ name: "projects/abc123" })` → note the prefix here, unlike every call in the next steps. Returns project info including screen instances, e.g. `{ id: "inst-1", sourceScreen: "scr-1" }` and `{ id: "inst-2", sourceScreen: "scr-2" }`.
3. `list_design_systems({ projectId: "abc123" })` → bare id again → returns an existing asset, e.g. `assets/ds456`.
4. `apply_design_system({ projectId: "abc123", assetId: "ds456", selectedScreenInstances: [{ id: "inst-1", sourceScreen: "scr-1" }, { id: "inst-2", sourceScreen: "scr-2" }] })` — note `assetId` is bare (`ds456`, not `assets/ds456`), and the instances come verbatim from step 2, not reconstructed from a screen id.

If step 4 instead used `update_design_system`, the same asset would be addressed as `name: "assets/ds456"` — prefixed, because that tool's parameter is `name`, not `assetId`.

## Common mistakes to avoid

- Passing a bare id to `get_project`, `get_screen`, or `delete_project` — these three are the exceptions that want the `projects/...` (or `projects/.../screens/...`) prefix; every other tool in the table wants a bare id.
- Passing a prefixed `assets/{id}` to `apply_design_system`'s `assetId` parameter — it wants the bare id, even though the sibling tool `update_design_system` wants the same asset prefixed under `name`.
- Using a screen id from `list_screens` in place of a screen instance id for `apply_design_system` or `create_design_system_from_design_md` — instance ids only come from `get_project`.
- Calling `create_project` before checking `list_projects`, resulting in duplicate projects with similar titles.
- Calling `delete_project` on the strength of an instruction given earlier in the conversation, without a fresh, explicit confirmation naming the specific project.
- Treating a stale `list_screens` or `get_project` result as current after any generation, edit, or variant call — re-fetch before relying on ids again.
