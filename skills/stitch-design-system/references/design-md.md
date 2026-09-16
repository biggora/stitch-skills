# DESIGN.md Pipeline — Reference

Deep-dive on Path B from `SKILL.md`: turning a written design document into a Stitch design system via `upload_design_md` → `create_design_system_from_design_md`.

## What belongs in a DESIGN.md

Stitch reads the document as prose plus structure, not as a token schema — so write it the way a designer would brief another designer, with concrete, specific language. A DESIGN.md that Stitch can act on well typically has:

- **A brand summary** — one or two sentences on who the product is for and the feeling the UI should evoke.
- **Color** — the actual brand colors (hex if you have them) and their intended roles (primary action, secondary, background, surface, error/warning if distinct). Say whether the product is light-mode, dark-mode, or both.
- **Typography** — named fonts (or font *characteristics* — "a warm literary serif for headlines" — if no font is chosen yet) for headline and body roles, and how sizes should scale (e.g. "generous heading sizes, compact body text").
- **Shape and spacing** — corner roundness intent ("soft, rounded cards" vs. "sharp, squared-off"), and density ("airy with generous whitespace" vs. "compact, information-dense").
- **Component tone** — how buttons, cards, and inputs should feel (flat vs. elevated/shadowed, bordered vs. borderless, filled vs. outlined).
- **Anything explicitly off-limits** — colors, fonts, or patterns the brand avoids, if relevant.

Keep it as prose organized under headings — Stitch parses meaning, not a fixed key-value format. Vague adjectives with no specifics ("modern," "clean," alone) produce vague results; pair every adjective with a concrete cue (a hex, a font name, a reference product).

### Example DESIGN.md

```markdown
# Brightleaf — Design Guidelines

Brightleaf is a budgeting app for freelancers who find traditional finance
apps intimidating. The UI should feel calm, trustworthy, and a little warm —
closer to a well-designed notebook than a bank's dashboard.

## Color

- Primary: #1D7A5F (a deep, confident green — used for primary actions,
  active states, and positive balances)
- Secondary: #E8DCC8 (warm parchment — used for card backgrounds and
  section dividers)
- Accent: #D97706 (amber — used sparingly, for warnings and due-soon states)
- Negative balances and errors use a muted brick red, never a harsh red.
- Light mode only for now; no dark mode variant is planned this quarter.

## Typography

- Headlines: a warm, literary serif — something in the spirit of Newsreader
  or Literata. Headlines should feel written, not corporate.
- Body and UI text: a clean, highly legible sans-serif (Inter or similar).
  Never use the headline serif below 16px.
- Numbers (balances, transaction amounts) should be set in a tabular
  numeric style and slightly larger than surrounding body text.

## Shape and spacing

- Corners are soft but not pill-shaped — cards and buttons should read as
  "rounded rectangle," not "capsule."
- Generous whitespace. This app should never feel like a spreadsheet.
- Cards sit on the parchment secondary color with a very subtle shadow,
  not a hard border.

## Components

- Buttons: filled for primary actions, outlined for secondary. No ghost
  buttons on the main dashboard — every actionable button should be visibly
  a button.
- Inputs: bottom-border style rather than fully boxed, to keep forms
  feeling light.
- Avoid iconography that looks like generic finance clip art (coins,
  piggy banks). Prefer simple line icons.

## What to avoid

- No harsh reds or alarm-style UI patterns, even for warnings.
- No dense data tables on primary screens — summarize, don't dump numbers.
- Avoid anything that looks like a spreadsheet or a bank's legacy portal.
```

This is roughly the right length for a small product: specific enough to constrain real decisions, short enough to read in under two minutes.

## Base64 encoding the DESIGN.md

`upload_design_md` takes `designMdBase64` — the base64 of the file's raw bytes, **not** wrapped or chunked into multiple lines, and the decoded content **must be valid UTF-8**. Producing the encoding incorrectly (wrapped lines, wrong encoding, a copy-paste that drops a character) is the most common way this path silently fails, since the tool will reject invalid UTF-8 after decoding.

**Linux / macOS (GNU coreutils, most Linux distros and modern macOS with GNU base64 installed):**

```bash
base64 -w 0 DESIGN.md
```

`-w 0` disables line wrapping — without it, GNU `base64` wraps at 76 characters by default, which breaks the parameter.

**macOS (BSD `base64`, the default on stock macOS without GNU coreutils):**

BSD `base64` has no `-w` flag and wraps by default, so strip the wrapping manually:

```bash
base64 -i DESIGN.md | tr -d '\n'
```

**Windows PowerShell:**

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("DESIGN.md"))
```

This reads the file as raw bytes and encodes in one call — inherently a single line, no extra flags needed.

In every case: verify the source file is saved as UTF-8 before encoding (not UTF-16 or a platform-specific codepage — this matters especially on Windows, where some editors default to UTF-16), and confirm the resulting string has no embedded newlines before passing it as `designMdBase64`.

## The full two-call sequence

1. **Encode** the DESIGN.md as described above.
2. **Call `upload_design_md`** with `projectId` (bare id) and `designMdBase64`. This registers the document against the project and returns a screen instance representing it — capture that instance's `id` and `sourceScreen` from the response; you need both for the next call.
3. **Call `create_design_system_from_design_md`**, immediately, with:
   - `projectId` — same bare id.
   - `selectedScreenInstance` — the exact `{id, sourceScreen}` object returned by `upload_design_md` in step 2, not a value from `get_project` or `list_screens`.
   - `deviceType` — optional; one of `DEVICE_TYPE_UNSPECIFIED | MOBILE | DESKTOP | TABLET | AGNOSTIC` (see `references/theme-enums.md`). Omit it or use `AGNOSTIC` unless the brief is specific to one device class.

Worked example:

```json
// Step 2
{
  "tool": "mcp__stitch__upload_design_md",
  "input": {
    "projectId": "a1b2c3d4",
    "designMdBase64": "IyBCcmlnaHRsZWFmIOKAlCBEZXNpZ24gR3VpZGVsaW5lcw...[truncated]"
  }
}

// Response includes something like:
// { "selectedScreenInstance": { "id": "inst_7f3a2b", "sourceScreen": "projects/a1b2c3d4/screens/scr_design_md_001" } }

// Step 3 — immediately after, using that exact instance
{
  "tool": "mcp__stitch__create_design_system_from_design_md",
  "input": {
    "projectId": "a1b2c3d4",
    "selectedScreenInstance": {
      "id": "inst_7f3a2b",
      "sourceScreen": "projects/a1b2c3d4/screens/scr_design_md_001"
    },
    "deviceType": "AGNOSTIC"
  }
}
```

The resulting design system is a normal Stitch design-system asset from this point on — it can be listed via `list_design_systems`, updated via `update_design_system`, and applied to other screens via `apply_design_system`, exactly as a structured-token system created through Path A.

## DESIGN.md vs. `theme.designMd`

These are two different things and it's easy to conflate them:

- **A DESIGN.md file, uploaded via `upload_design_md`**, is the *source document* for the whole two-call generation pipeline above. Stitch parses it to derive fonts, colors, and shape tokens automatically.
- **`theme.designMd`**, a field on the `DesignTheme` object used by `create_design_system` / `update_design_system` (Path A), is just a markdown *description string attached to an already-structured theme* — it does not get parsed to derive tokens. Use it to carry human-readable notes or rationale alongside explicit `headlineFont`, `bodyFont`, `customColor`, etc. values you're already setting directly.

Rule of thumb: if you want Stitch to *derive* the tokens from prose, upload a real DESIGN.md through Path B. If you already know the exact tokens and just want to attach descriptive notes to them, set `theme.designMd` as a supplementary field on a Path A call.
