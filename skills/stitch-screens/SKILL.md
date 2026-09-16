---
name: stitch-screens
description: Use when the user wants to generate a screen, create a mockup, produce a new page design, add a screen to a Google Stitch project, or otherwise turn a text description into a UI screen through the Stitch MCP server. Applies when the user says things like "generate a screen", "create a mockup", "make a new page design", "add a screen for...", "design a login page in Stitch", "build me a dashboard mockup", "generate a few screens for this app", or asks Stitch to produce a first draft of any UI screen from a text prompt. For changing a screen that already exists, use stitch-iterate.
---

# Stitch Screens

Owns screen generation from text for the Google Stitch MCP server (`stitch`, tools exposed as `mcp__stitch__<tool>`). This skill covers `generate_screen_from_text` only. For changing a screen that already exists, use `stitch-iterate` if installed, or call `edit_screens` (`projectId`, `selectedScreenIds`, `prompt`) for refinements and `generate_variants` (`projectId`, `selectedScreenIds`, `prompt`, `variantOptions`) for alternatives directly — both are long-running and the protocol below applies to them unchanged.

## Pre-flight: resolve before you generate

Do not call `generate_screen_from_text` until these are settled:

1. **`projectId`.** Required, bare id with no `projects/` prefix (e.g. `'4044680601076201931'`). If you don't already have it, resolve it with `list_projects` (and `create_project` only if nothing existing fits) — see `stitch-projects` if that skill is installed, or call those tools directly otherwise.
2. **The design system `assetId`.** Optional on the tool signature, but treat it as required in practice. Pass it as `designSystem: "assets/{id}"` — **with** the `assets/` prefix, unlike almost every other id this tool touches. Obtain the id via `get_project` or `list_design_systems`. The tool's own documentation states a design system "should always be configured for design consistency," and skipping it is not free: a screen generated without one drifts stylistically from the rest of the project, and the only fix afterward is a full `apply_design_system` pass, which itself needs screen **instance** ids from `get_project` — a strictly more expensive path than just passing `designSystem` up front. If the user explicitly says the screen is disposable or a one-off throwaway, it's fine to omit `designSystem` and accept the default design system; otherwise resolve it first.
3. **`deviceType`.** Optional: `DEVICE_TYPE_UNSPECIFIED | MOBILE | DESKTOP | TABLET | AGNOSTIC`. Decide this once per project and keep it consistent across every screen you generate for that project — mixing `MOBILE` and `DESKTOP` screens within one project produces a visually incoherent set. If the user hasn't said, ask, or infer it from context (e.g. "an app screen" leans `MOBILE`, "an admin panel" leans `DESKTOP`) rather than leaving it unspecified by default.

Before writing any non-trivial prompt for `generate_screen_from_text`, read `references/prompting.md`. It has the full anatomy of an effective Stitch prompt, a weak-vs-strong comparison table, and ready-to-adapt example prompts for common screen types — do not improvise prompt structure from scratch when that reference exists.

## The long-running protocol

`generate_screen_from_text` is long-running and its own documentation is explicit about how to handle that. Three rules, non-negotiable:

1. **Never retry.** The tool's documentation states outright: "This action can take a few minutes to complete. Please be patient. DO NOT RETRY." A slow response is expected behavior, not a failure.
2. **On a timeout, poll — don't retry.** "If the tool fails with a timeout, don't retry. Instead, try to get the screen with `get_screen` method every 30 seconds for up to 10 times before giving up."
3. **A connection error is not a failure.** "If the tool call fails due to connection error, the generation process may still succeed. Please try to get the screen with `get_screen` method later." Treat an apparent failure as unknown status, not as confirmed failure, until you've checked.

**The corollary that matters most in practice:** firing a second `generate_screen_from_text` call after what looks like a failed or hung first call is how users end up with duplicate screens they didn't ask for and burned generation quota for nothing. If a call seems to have failed, your very next action is to verify via polling, never to re-issue the same generation.

### A concrete polling procedure

When a `generate_screen_from_text` call times out or drops the connection, follow this literally:

1. Once, call `list_screens` with the same `projectId` to find the id of the screen the call was creating, if you do not already have it.
2. Call `get_screen` (`name: "projects/{project}/screens/{screen}"` — prefixed, unlike `list_screens`) and inspect whether it has finished generating and has real content.
3. If it has not, wait roughly 30 seconds and call `get_screen` again. Count only `get_screen` calls toward the limit; stop after 10.
4. If after 10 `get_screen` calls there is still no confirmed result, stop and report honestly that generation could not be confirmed — do not claim success, and do not re-generate.

You cannot literally sleep between polls. Use whatever wait or monitoring mechanism your host environment provides for spacing out repeated checks, or, if no such mechanism exists, tell the user plainly that you are checking on a roughly 30-second cadence and proceed to the next check when you actually take it. Never fabricate the passage of time, never claim to have polled when you haven't actually called `get_screen` or `list_screens`, and never report a generation result you have not actually observed in a tool response.

Attempts are only meaningful if real time passes between them. If your environment has no wait mechanism, do not spin through the ten checks back to back — that reports a false failure on a generation that is still running. Instead, take one check, tell the user plainly that the generation is still in progress and roughly how long it has been running, and ask them to prompt you to check again. Resuming the count in a later turn is correct; exhausting it in four seconds is not.

## Handling `output_components`

`generate_screen_from_text` can return `output_components` alongside the generated screen. The tool's documentation is specific about what to do with each kind of content it may contain:

- **If it contains text**, return that text to the user — it's meant to be seen, not silently absorbed.
- **If it contains suggestions** (for example, something like "Yes, make them all"), present these suggestions to the user as an explicit choice. Do not act on a suggestion yourself.
- **Only if the user accepts one of the suggestions**, call `generate_screen_from_text` again with `prompt` set to the accepted suggestion's text.

Never auto-accept a suggestion on the user's behalf, even if it looks obviously beneficial or the user seems likely to want it. The decision belongs to the user; your job is to surface it clearly and wait.

## Model choice: `modelId`

Optional enum: `MODEL_ID_UNSPECIFIED | GEMINI_3_8_FLASH | GEMINI_3_5_FLASH_LITE`. The available documentation describes `GEMINI_3_5_FLASH_LITE` as the smaller/faster option and `GEMINI_3_8_FLASH` as the stronger one, but gives no quality benchmark distinguishing them for screen generation specifically — be honest about that gap rather than asserting a quality difference you can't verify. Default to omitting `modelId` and letting Stitch pick, unless the user explicitly asks for faster generation (in which case `GEMINI_3_5_FLASH_LITE` is the reasonable choice) or explicitly asks for the stronger model by name.

## Multi-screen flows

In Codex, the Claude Code agent at `agents/stitch-batch-generator.md` is not registered as a custom agent. Run the multi-screen procedure below in the current conversation. Keep a ledger with each requested screen's label, observed screen id, and status (`pending`, `generating`, `succeeded`, `failed`, or `unconfirmed`), updating it after each actual result. Return that ledger as a table and attribute any `output_components` text or suggestions to the screen that produced them. If interrupted, report the observed status of the in-flight screen and leave unstarted screens `pending`.

When a request calls for more than one screen (e.g. "generate a login screen and a dashboard"), generate them one at a time — there is no batch form of `generate_screen_from_text`. For each call:

- Reuse the same `designSystem` and `deviceType` resolved during pre-flight so the whole set stays visually consistent.
- Apply the long-running protocol above independently to each call; do not assume that because one screen generated cleanly, the next will too.
- Keep a running list of the screen ids returned (or discovered via `list_screens` while polling), since later steps — edits, variants, or handing off to `stitch-iterate` — need those ids and they are not otherwise recoverable except by re-listing.

If the user's request is really "generate N variations of the same screen" rather than N distinct screens, that's a different tool: point them at `generate_variants` (covered by `stitch-iterate`), not repeated calls to `generate_screen_from_text`.

Report progress honestly as you go through a multi-screen sequence. If screen 2 of 4 fails to confirm after the full polling procedure, say so and ask the user whether to keep going with the remaining screens or pause — don't silently skip it and continue as if nothing happened, and don't claim all four succeeded when one is still unconfirmed.

## Confirming a screen actually generated

Before telling the user a screen is ready, make sure you've actually seen evidence of that in a tool response — either the direct return value of `generate_screen_from_text`, or a `get_screen`/`list_screens` result obtained while polling. A few concrete situations worth calling out:

- If the call returns normally with a screen result, that's your confirmation — no further polling needed.
- If you only got there via the timeout-polling procedure, the confirmation is the `get_screen` response showing real, finished content, not merely the screen id existing in `list_screens`. A screen id can appear in `list_screens` while generation is still in progress; `get_screen` is what tells you whether it's actually done.
- Never tell the user a screen "should be ready by now" as a substitute for checking. If you haven't called `get_screen` (or the original call hasn't returned), you don't know that yet.

## Anti-patterns to avoid

- **Re-generating instead of polling** after a timeout or connection error — creates duplicate screens and wastes quota, and directly contradicts the tool's own "DO NOT RETRY" instruction.
- **Skipping `designSystem`** without the user having said the screen is disposable — cheaper up front to configure it than to retrofit consistency later via `apply_design_system`.
- **Switching `deviceType` partway through a project's screens** without the user asking for a device-specific variant — breaks visual consistency across the set.
- **Auto-accepting an `output_components` suggestion** on the user's behalf instead of presenting it as a choice.
- **Writing a one-line prompt** for anything beyond the most trivial screen — consult `references/prompting.md` first; a vague prompt produces a vague screen, and fixing it afterward costs an `edit_screens` round trip that a better first prompt would have avoided.

## Worked example

```json
{
  "tool": "generate_screen_from_text",
  "arguments": {
    "projectId": "4044680601076201931",
    "prompt": "A mobile onboarding screen for a budgeting app...",
    "designSystem": "assets/15996705518239280238",
    "deviceType": "MOBILE"
  }
}
```

Note the asymmetry: `projectId` is bare, `designSystem` is prefixed with `assets/`. Getting this backwards is the most common mistake when calling this tool — double-check both before sending the call.
