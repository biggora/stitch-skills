# stitch-skills

Codex and Claude Code skills for designing UI with **Google Stitch** through the Stitch MCP server.

Stitch generates screens and design systems from text. The MCP surface is small but full of
sharp edges: identifiers are prefixed in some calls and bare in others, screen *instance* ids
are not screen ids, two tool pairs must be called back-to-back or the result is invisible, and
generation is a multi-minute operation where retrying is explicitly forbidden. These skills
encode those rules so the model gets them right the first time.

## Skills

| Skill | Use it for |
|---|---|
| `stitch-workflow` | Entry point. The end-to-end sequence, session state, and routing to the other skills. |
| `stitch-projects` | Creating and inspecting projects; the complete identifier-format reference. |
| `stitch-design-system` | Theme tokens and fonts; the `DESIGN.md` pipeline **in both directions**, including export. |
| `stitch-screens` | Generating screens from prompts, and surviving the long-running generation protocol. |
| `stitch-iterate` | Targeted edits (`edit_screens`) and exploring alternatives (`generate_variants`). |
| `stitch-components` | Downloading a screen HTML and slicing it into reusable components. |
| `stitch-storybook` | Scaffolding Storybook, registering components as stories, composing pages. |
| `stitch-to-code` | Turning a screen and its tokens into code that matches your project stack. |

### The design-to-Storybook pipeline

The skills chain into one route out of Stitch:

```
Stitch project ──> DESIGN.md + design tokens      (stitch-design-system, export)
               └─> screen HTML ──> components     (stitch-components)
                                └─> stories ──> pages  (stitch-storybook)
```

Two facts drive it, both verified against the live API: `get_screen` returns a
`htmlCode.downloadUrl` rather than inline markup, and the downloaded document is
Tailwind-based with a resolved palette in an inline `tailwind.config`. The design system
also exposes resolved tokens — `namedColors`, `typography`, `spacing` and real font
family names — so tokens are read, not guessed.

If you store the key in a project `.env`, load `STITCH_API_KEY` into the host's process
environment before launching it. The plugin does not automatically load `.env` files.

## Installation

Both hosts install straight from this repository — no checkout required. `biggora/stitch-skills`
is the GitHub shorthand, not a local path.

### Codex

```bash
codex plugin marketplace add biggora/stitch-skills
codex plugin add stitch-skills@stitch
```

Verified with Codex CLI 0.154.0: these GitHub commands install the plugin, and Codex
loads all six bundled skills from the installed cache. If your host does not expose
the install command, open its plugin directory, select the **Google
Stitch** marketplace, and install **stitch-skills**. Start a new conversation after
installation so its skills and MCP tools are loaded.

Live Google Stitch access was also verified with a key loaded from `.env`: discovery
of all 15 MCP tools and a successful read-only `list_projects` call. Screen generation
was not exercised by this check.

The [Codex marketplace](.agents/plugins/marketplace.json) exposes the full toolkit as
one plugin, using the existing repository root and `skills/` directory. Individual
plugin entries below are specific to Claude Code. This uses the supported Codex
compatibility manifest format; see the [OpenAI packaging documentation](https://developers.openai.com/plugins/build/plugins).

### GitHub Copilot CLI

Add the marketplace once:

```text
/plugin marketplace add biggora/stitch-skills
```

Then install the plugin:

```text
/plugin install stitch-skills@stitch
```

The root `.mcp.json` remains canonical. If MCP is not auto-loaded, configure `/mcp` with
URL `https://stitch.googleapis.com/mcp` and header `X-Goog-Api-Key: ${STITCH_API_KEY}`.

### Claude Code

Add the marketplace once:

```bash
/plugin marketplace add biggora/stitch-skills
```

Then install **everything**:

```bash
/plugin install stitch-skills@stitch
```

…or **only the skills you want**:

```bash
/plugin install stitch-screens@stitch
```

Every entry ships the same `.mcp.json`, so the Stitch server is configured whichever you pick.
Installing more than one à-la-carte entry is fine — they share one underlying directory, so
nothing is duplicated on disk.

### npx skills (any agent)

Skills also install individually via the open-source [`skills`](https://github.com/vercel-labs/skills)
CLI, straight from this repository — no checkout required:

```bash
npx skills add https://github.com/biggora/stitch-skills --skill stitch-screens
```

`--skill` repeats for more than one:

```bash
npx skills add https://github.com/biggora/stitch-skills --skill stitch-design-system --skill stitch-screens
```

Skills land in `./.claude/skills/<name>/` by default, `references/` included; add `-g` to
install to `~/.claude/skills/` instead. This is the route for agents beyond Claude Code and
Codex — Cursor, Gemini CLI, Zed, opencode, and 75+ others — since the tool detects the host
itself (its `--agent` flag, if you set it explicitly, wants `claude-code`, not `claude`).

> **Configure MCP yourself.** `npx skills` installs skill files only, not `.mcp.json`. Every
> skill here calls `mcp__stitch__*` tools, so without the `stitch` MCP server configured — URL
> `https://stitch.googleapis.com/mcp`, header `X-Goog-Api-Key: ${STITCH_API_KEY}` — an installed
> skill has instructions but nothing to act with. The marketplace and Codex routes above
> configure this for you automatically; this one does not.

### Local development

Only for working on the plugin itself. Clone the repository and load the development
copy from its root; regular installation uses the GitHub commands above:

```bash
git clone https://github.com/biggora/stitch-skills.git
cd stitch-skills
claude --plugin-dir .
codex plugin marketplace add .
```

### Manual, single-skill install

The same idea without Node tooling. It needs a clone (`npx skills` above works straight
from the URL and is the better-supported option), and keeps no lock file or update record.
From the repository root, each self-contained skill directory can be copied into your
personal skills directory:

```bash
cp -r skills/stitch-screens ~/.claude/skills/stitch-screens
```

For Codex, copy it to `~/.agents/skills/stitch-screens` instead. Same caveat as `npx skills`:
this brings no `.mcp.json` — configure the `stitch` MCP server yourself.

## Per-project settings

Download the template from this repository into the project where you use Stitch.
No plugin checkout is required:

```bash
# Codex
mkdir -p .codex
curl -fsSL https://raw.githubusercontent.com/biggora/stitch-skills/main/examples/stitch.local.md -o .codex/stitch.local.md

# Claude Code
mkdir -p .claude
curl -fsSL https://raw.githubusercontent.com/biggora/stitch-skills/main/examples/stitch.local.md -o .claude/stitch.local.md
```

Fill in the project id and design system id so you stop repeating them in every request. The
file holds no secrets — the API key stays in the environment. Add `.codex/*.local.md`
or `.claude/*.local.md` to the consuming project's `.gitignore`. Codex also reads an
existing `.claude/stitch.local.md` when its `.codex/` counterpart is absent; if both
exist, each host uses its own file.

## Usage

The skills activate on their own when a request matches. Nothing to invoke manually:

- *"Build me a design system for a fintech app — indigo, geometric, dark mode"* → `stitch-design-system`
- *"Generate a mobile onboarding screen"* → `stitch-screens`
- *"Show me three variants with a different color scheme"* → `stitch-iterate`
- *"Implement this screen as a React component"* → `stitch-to-code`

## Caveats

- **Generation is slow.** Screen generation, edits, and variants take minutes. The skills never
  retry — they poll `get_screen` instead, per the Stitch tool contract. Retrying is how you end
  up with duplicate screens and burned quota.
- **A connection error is not a failure.** The generation may have succeeded server-side. The
  skills verify with `list_screens` before concluding anything.
- **A Stitch screen is a design, not a spec.** `stitch-to-code` will not invent business logic,
  routes, or API calls that you did not ask for.

## License

MIT
