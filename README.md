# Vicuna plugin: Muse harness

Adds the **Muse** coding agent to [Vicuna](https://github.com/anslwy/vicuna) as a Work harness.
Vicuna runs `muse acp` and talks to it over the [Agent Client Protocol](https://agentclientprotocol.com) (ACP),
the same way it drives its built-in ACP harnesses (OpenCode, Antigravity, fx).

## Install

1. Install the CLI. See "Install the CLI" below.
2. In Vicuna: **Settings → Plugins → Explore → Muse Harness → Install** (or paste `anslwy/vicuna-plugin-muse`).
3. Vicuna shows exactly which command the harness will run (`muse acp`). Click **Allow and enable**.
4. Pick **Muse** in the Work harness picker. Models come from the agent itself.

### Install the CLI

This plugin does **not** ship any program. You install `muse` yourself, sign in once with `muse login`,
and make sure it is on your PATH. Vicuna finds it the way your terminal does.

## Template: writing a harness plugin

Any CLI that serves ACP over stdio can become a harness. A plugin is just a `vicuna-plugin.json`:

```json
{
  "id": "my-agent",
  "name": "My Agent Harness",
  "version": "1.0.0",
  "description": "What it adds.",
  "harnesses": [
    {
      "id": "my-agent",
      "label": "My Agent",
      "acp": { "command": "my-agent", "args": ["acp"] },
      "models": [{ "id": "fast-1", "label": "Fast" }],
      "installHint": "Install my-agent and sign in once.",
      "installUrl": "https://example.com/install"
    }
  ]
}
```

| Field | Rules |
|---|---|
| `id` | Lowercase letters, digits and dashes, up to 40 characters, unique within the plugin. The engine id becomes `plugin:<plugin-id>/<harness-id>`. |
| `label` | 1–40 characters, shown in the harness picker. |
| `acp.command` | **A bare program name only** (letters, digits, `. _ + -`). No `/`, `\`, `:` or `..`, and it can't start with `.` or `-`. Vicuna looks it up on the user's PATH (plus nvm/fnm/volta/asdf/mise bins). Absolute and relative paths are refused, so a plugin can't point at a file it ships or at some other location. |
| `acp.args` | Up to 32 strings of up to 256 characters each, passed straight to the process. **No shell and no interpolation.** Control characters and `` ` $ ; \| & < > " ' \ `` are refused. |
| `models` | Optional, up to 64 `{id, label?}`. If you leave it out, the picker lists the models the agent reports in ACP `session/new` (`configOptions` model select or `models.availableModels`). |
| `installHint` / `installUrl` | Shown when the command isn't on PATH. `installUrl` must be https. |

A plugin can declare up to 4 harnesses.

### Consent

A harness stays **disabled** until the user approves the exact command and args in a dialog. Vicuna also tells
the user that the agent works in the session's workspace with the same powers as the built-in harnesses.
Consent is stored on the Vicuna server next to the plugin's `install.json`, and the server checks it every time it
starts the agent. If an upgrade changes `command` or `args`, the harness is disabled again until the user approves
the new command. Uninstalling the plugin removes the harness, and sessions that used it show a message asking you
to pick another harness.

### What Vicuna does with an ACP agent

`initialize` → `session/new` (cwd = the session workspace, plus the user's MCP servers and Vicuna's own
`vicuna-mcp`) or `session/load` to resume → `session/set_config_option {configId: "model"}`
(falling back to `session/set_model`) → `session/prompt`, streaming `session/update`. Permission requests
are approved automatically or shown as cards, depending on the user's setting. Vicuna runs one process per
workspace, started in that folder.
