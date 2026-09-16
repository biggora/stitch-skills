---
name: stitch-iterate
description: Refines Google Stitch screens that already exist — editing a screen in place with edit_screens, or generating alternative versions with generate_variants. Applies when the user wants to change an existing Stitch screen, edit the design, try another version, see variants, explore alternatives, or refine a screen Stitch already generated, and when the user says things like "change the screen", "edit the design", "make it darker", "try another version", "show me variants", "explore alternatives", "tweak this screen", "add a button here", "regenerate this section", or "give me a few options for this layout" — that is, an existing Stitch screen refined, varied, or reworked rather than created from nothing.
---

# Stitch Iterate

Owns refinement of screens that already exist in a Google Stitch project (MCP server `stitch`, tools exposed as `mcp__stitch__<tool>`): `edit_screens` and `generate_variants`. This skill is fully self-contained — it does not depend on `stitch-screens` or any other skill being installed, though it may be used together with `stitch-screens` (which owns generating brand-new screens with `generate_screen_from_text`).

## Decision rule: which tool to use

Three ways to change a screen exist; pick based on how settled the direction of the change is.

| Situation | Use |
|---|---|
| The change is known and directed — you know exactly what should be different | `edit_screens` |
| The direction isn't decided yet — you want to compare alternatives before committing | `generate_variants` |
| The screen's fundamental purpose changed (it's no longer the same screen, just similar) | Full regeneration from scratch (`generate_screen_from_text`, not covered by this skill) |

If in doubt between the first two: a request phrased as an instruction ("make the header sticky," "add a filter bar," "make it darker") is `edit_screens`. A request phrased as exploration ("what could this look like," "give me some options," "show me variants," "try a different layout") is `generate_variants`. When the user's phrasing is genuinely ambiguous, ask which they want rather than guessing — the two tools have very different effects on the project (one mutates in place, the other adds new alternatives).

## `edit_screens` in depth

- `projectId` — **required**, bare id, no `projects/` prefix.
- `selectedScreenIds` — **required**, array of bare screen ids (no `screens/` prefix). Example: `['98b50e2ddc9943efb387052637738f61']`.
- `prompt` — **required**.
- `deviceType`, `modelId` — optional, same enums as generation (`deviceType`: `DEVICE_TYPE_UNSPECIFIED | MOBILE | DESKTOP | TABLET | AGNOSTIC`; `modelId`: `MODEL_ID_UNSPECIFIED | GEMINI_3_8_FLASH | GEMINI_3_5_FLASH_LITE`).

**One prompt is applied across every id in `selectedScreenIds`.** This is a deliberate lever, not just a batching convenience: use it when you want the *same* change applied consistently across a heterogeneous set of screens, e.g. "add a persistent bottom navigation bar with Home, Search, and Profile tabs to every screen" run once against all of a project's screen ids, rather than one `edit_screens` call per screen with slightly different wording each time (which invites the inconsistency a shared navigation bar is supposed to prevent).

**The failure mode to watch for is the opposite case:** a prompt written with one specific screen in mind, then broadcast across a set of screens that don't all share that context, produces poor results on the screens it wasn't really written for. "Change the hero image to show the new product" makes sense for a landing page but will confuse or misfire on a settings screen swept into the same `selectedScreenIds` array. Before calling `edit_screens` with more than one id, check that the prompt genuinely applies, unmodified, to every screen in the list — if it doesn't, split the call into separate `edit_screens` calls per screen (or per subgroup) instead.

### Writing surgical edit prompts

An edit prompt should read like a change order, not a fresh screen brief:

- **Name the element.** "The primary call-to-action button in the hero section" beats "the button."
- **Name the change.** State the concrete new state, not just the problem — "increase the button's size and change its label to 'Get Started Free'" beats "make the button better."
- **State what must remain untouched.** If the rest of the screen should not shift, say so explicitly: "Only change the button; leave the layout, other copy, and colors exactly as they are." Without this, an edit can bring unrequested changes elsewhere on the screen.

Weak: *"Improve the settings screen."*
Strong: *"On the settings screen, add a new toggle row labeled 'Two-Factor Authentication' directly below the existing 'Password' row, using the same row style (leading icon, label, trailing control) as the other rows in the Account section. Do not change any other section or row."*

## `generate_variants` in depth

- `projectId` — **required**, bare id.
- `selectedScreenIds` — **required**, array of bare screen ids.
- `prompt` — **required**.
- `variantOptions` — **required** object (required as an object, though every field within it has a default):
  - `variantCount` — optional int, **1–5**, default **3**. More variants cost more time to generate — do not default to the maximum just because it's available; ask for as many as the user actually plans to compare.
  - `creativeRange` — optional: `CREATIVE_RANGE_UNSPECIFIED | REFINE | EXPLORE | REIMAGINE`.
  - `aspects` — optional array: `VARIANT_ASPECT_UNSPECIFIED | LAYOUT | COLOR_SCHEME | IMAGES | TEXT_FONT | TEXT_CONTENT`. Documented behavior: "If empty, all aspects may be varied."
- `deviceType`, `modelId` — optional, same enums as above.

### Mapping intent to `creativeRange` and `aspects`

| User intent | `creativeRange` | `aspects` |
|---|---|---|
| "Try slightly different shades of this same color scheme" | `REFINE` | `["COLOR_SCHEME"]` |
| "Show me a few different color palettes for this screen" | `EXPLORE` | `["COLOR_SCHEME"]` |
| "What if the layout were totally different" | `REIMAGINE` | `["LAYOUT"]` |
| "Try a couple of alternate headlines/copy options" | `EXPLORE` | `["TEXT_CONTENT"]` |
| "Give me a few different fonts to compare" | `REFINE` or `EXPLORE` | `["TEXT_FONT"]` |
| "Try different hero images" | `EXPLORE` | `["IMAGES"]` |
| "I have no idea what direction to take this, surprise me" | `REIMAGINE` | leave empty (all aspects) |
| "Just polish this, nothing dramatic" | `REFINE` | leave empty, or scope to the specific aspect if one is implied |

Documented meanings for `creativeRange`: `REFINE` — "Subtle refinements, closely adhering to original"; `EXPLORE` — "Balanced exploration. Default."; `REIMAGINE` — "Radical explorations, fundamentally challenging the original."

**Constrain `aspects` whenever the user's intent is about one dimension.** Leaving `aspects` empty means, per the tool's own documentation, "all aspects may be varied" — every variant can differ in layout, color, images, font, and text simultaneously. That produces variants that are hard to compare against each other, because each one differs from the original (and from its siblings) in several dimensions at once, so it's unclear which change is responsible for which effect. A focused array like `["COLOR_SCHEME"]` isolates the one thing being compared and yields variants that are actually useful side by side. Only leave `aspects` empty when the user genuinely wants open-ended exploration across every dimension at once.

## The long-running protocol (condensed)

Both `edit_screens` and `generate_variants` are long-running, same as screen generation:

- **Never retry** a call that seems slow or hung. Patience is expected, not a sign of failure.
- **On a timeout, poll instead of retrying**: call `get_screen` (`name: "projects/{project}/screens/{screen}"`, prefixed) every 30 seconds, up to 10 attempts, to check whether the edit or variant generation actually completed.
- **A connection error is not a confirmed failure.** The process may have succeeded server-side even though the response didn't come back — verify with `get_screen` or `list_screens` before concluding anything, and before considering a retry.
- You cannot literally sleep between polls; use whatever wait/monitoring mechanism your host provides, or tell the user plainly you're checking periodically. Never claim to have polled, waited, or observed a result you have not actually observed via a real tool call.
- If 10 attempts pass with nothing confirmed, stop and report honestly that the result could not be confirmed, rather than guessing at an outcome.

## Choosing which screens to operate on

- Use `list_screens` (`projectId`, bare) to enumerate the screens available in a project and get their bare ids for `selectedScreenIds`.
- Use `get_screen` (`name: "projects/{project}/screens/{screen}"`, prefixed) to confirm a specific screen's identity and current content before mutating it — especially when a screen id came from earlier in a long conversation and might be stale, or when the user refers to "the login screen" and more than one screen could plausibly match that description.
- **`edit_screens` mutates in place.** There is no separate "preview" step — once it runs, the targeted screens are changed. If the target screen is ambiguous (multiple candidates, or the user's description could match more than one screen), confirm the exact screen id with the user before calling `edit_screens`, rather than guessing and mutating the wrong one.
- If the user might want to keep the original screen alongside the new option, that's a signal to reach for `generate_variants` instead of `edit_screens` — variants are additional alternatives, not in-place replacements, so the original stays intact by default.

## Worked examples

**`edit_screens`** — applying one directed change across two screens:

```json
{
  "tool": "mcp__stitch__edit_screens",
  "input": {
    "projectId": "4044680601076201931",
    "selectedScreenIds": [
      "98b50e2ddc9943efb387052637738f61",
      "3fa1c9e2b8d74a0eaf6ce0f2a9d51234"
    ],
    "prompt": "Add a persistent bottom navigation bar with three tabs: Home, Search, and Profile, with Home shown as the active tab. Do not change anything else on the screen.",
    "deviceType": "MOBILE"
  }
}
```

**`generate_variants`** — exploring color-scheme alternatives for one screen:

```json
{
  "tool": "mcp__stitch__generate_variants",
  "input": {
    "projectId": "4044680601076201931",
    "selectedScreenIds": ["98b50e2ddc9943efb387052637738f61"],
    "prompt": "Explore alternate color schemes for this dashboard screen, keeping the layout and content exactly as they are.",
    "variantOptions": {
      "variantCount": 3,
      "creativeRange": "EXPLORE",
      "aspects": ["COLOR_SCHEME"]
    }
  }
}
```

Note the identifier shapes: `projectId` and every entry in `selectedScreenIds` are bare, with no prefix, in both calls above. Only `get_screen` (used for polling or confirming identity, not shown in either call) needs the fully prefixed `projects/{project}/screens/{screen}` form.
