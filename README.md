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
| `stitch-design-system` | Theme tokens, fonts, color variants, roundness, and the `DESIGN.md` pipeline. |
| `stitch-screens` | Generating screens from prompts, and surviving the long-running generation protocol. |
| `stitch-iterate` | Targeted edits (`edit_screens`) and exploring alternatives (`generate_variants`). |
| `stitch-to-code` | Turning a screen and its tokens into code that matches your project's actual stack. |

Also included: a Claude Code `stitch-batch-generator` subagent for generating several screens
in sequence, and a settings template at
[`examples/stitch.local.md`](examples/stitch.local.md).
Codex runs multi-screen generation through `stitch-screens`, with a progress ledger in the
current conversation; it does not load the Claude Code agent.

## Prerequisites

A Google Stitch API key, exposed as an environment variable:

```bash
export STITCH_API_KEY="your-key-here"
```

On Windows (PowerShell):

```powershell
[Environment]::SetEnvironmentVariable("STITCH_API_KEY", "your-key-here", "User")
```

Claude Code's [`.mcp.json`](.mcp.json) reads the key from this variable. Codex's
[manifest](.codex-plugin/plugin.json) uses `env_http_headers` to read the same variable,
with a 600-second tool timeout for long-running generation. **No key is stored in this
repository.** Set the variable before starting your host, or the `stitch` MCP server will
fail to authenticate. In PowerShell, use `$env:STITCH_API_KEY = "your-key-here"` as well
if launching the host from the current shell; the persistent User setting applies to
new processes after their environment is refreshed.

## Installation

Both hosts install straight from this repository — no checkout required. `biggora/stitch-skills`
is the GitHub shorthand, not a local path.

### Codex

```bash
codex plugin marketplace add biggora/stitch-skills
codex plugin add stitch-skills@stitch
```

Verified with Codex CLI 0.154.0: the second command installs the plugin. If your
host does not expose that command, open its plugin directory, select the **Google
Stitch** marketplace, and install **stitch-skills**. Start a new conversation after
installation so its skills and MCP tools are loaded.

The [Codex marketplace](.agents/plugins/marketplace.json) exposes the full toolkit as
one plugin, using the existing repository root and `skills/` directory. Individual
plugin entries below are specific to Claude Code. This uses the supported Codex
compatibility manifest format; see the [OpenAI packaging documentation](https://developers.openai.com/plugins/build/plugins).

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

### Local development

Only for working on the plugin itself, from a clone of this repository. Use a real path
here — this is the one case where a local path is correct:

```bash
claude --plugin-dir /path/to/stitch-skills
codex plugin marketplace add /path/to/stitch-skills
```

### Manual, single-skill install

Each skill directory is self-contained and uses no plugin-relative paths, so it also works
copied straight into your personal skills directory:

```bash
cp -r skills/stitch-screens ~/.claude/skills/stitch-screens
```

For Codex, copy it to `~/.agents/skills/stitch-screens` instead.

You must then configure the `stitch` MCP server yourself — the manual copy does not bring
`.mcp.json` with it.

## Per-project settings

Copy the template into any project where you use Stitch:

```bash
# Codex
mkdir -p .codex
cp examples/stitch.local.md .codex/stitch.local.md

# Claude Code
mkdir -p .claude
cp examples/stitch.local.md .claude/stitch.local.md
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
