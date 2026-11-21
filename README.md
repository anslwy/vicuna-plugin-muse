# Vicuna plugin: Muse harness

Adds the **Muse** coding agent to [Vicuna](https://github.com/anslwy/vicuna) as a Work harness.
Vicuna runs `muse acp` and talks to it over the [Agent Client Protocol](https://agentclientprotocol.com) (ACP),
the same way it drives its built-in ACP harnesses (OpenCode, Antigravity, fx).

## Install

1. Install the CLI. See "Install the CLI" below.
2. In Vicuna: **Settings → Plugins → Explore → Muse Harness → Install** (or paste `anslwy/vicuna-plugin-muse`).
3. Vicuna shows exactly which commands the harness will run (`muse acp`, and `muse usage --json` for the Usage view). Click **Allow and enable**.
4. Pick **Muse** in the Work harness picker. Models come from the agent itself.
5. The Work header's usage meter (and the agents' `usage_status` tool) now has a **Muse** tab with your session and weekly allowance.

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
      "installUrl": "https://example.com/install",
      "usage": {
        "command": "my-agent",
        "args": ["usage", "--json"],
        "map": {
          "plan": "/plan",
          "note": "/message",
          "meters": [
            { "label": "Weekly", "usedPercent": "/weekly/used_percent", "resetsAt": "/weekly/resets_at" },
            { "each": "/models", "labelPath": "/name", "group": "Models", "used": "/requests", "limit": "/max_requests" }
          ]
        }
      }
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

### Usage (optional)

`usage` lets Vicuna show the harness's subscription allowance in its Usage view and in the agents' `usage_status`
tool, like the built-in Claude Code / Codex / Antigravity tabs. Vicuna runs `usage.command` + `usage.args` (same
rules as `acp`: a bare program on PATH, args list, no shell), reads at most 256 KB of stdout, stops it after 15 s,
and parses it as **JSON only**. No plugin code runs: the `map` says where the numbers are, as
[JSON pointers](https://www.rfc-editor.org/rfc/rfc6901) (`/a/b/0`).

| Field | Rules |
|---|---|
| `map.plan` / `map.note` | Optional pointers to a plan name and a short note (the note explains an empty result). |
| `map.meters` | 1–16 meters. Each needs `label` (fixed text) or `labelPath`, and **exactly one** of `usedPercent`, `remainingPercent`, or `used` + `limit`. |
| `resetsAt` | Optional. An ISO 8601 string or an epoch number (seconds, or milliseconds; Vicuna tells them apart by size). |
| `group` | Optional title; meters with the same group are shown together. |
| `each` | Optional pointer to an array: the meter repeats for every element, and its other pointers are relative to that element. |

Percentages and counts may be numbers or numeric strings. A meter whose value is missing is skipped. If no meter
has a value, the tab says the usage is unknown (with `note` as the reason when there is one). Never put account
emails or identifiers in the mapped fields: they would reach the Usage view. Vicuna never passes the account to agents.

### Consent

A harness stays **disabled** until the user approves the exact command and args (and the usage command, if any) in a dialog. Vicuna also tells
the user that the agent works in the session's workspace with the same powers as the built-in harnesses.
Consent is stored on the Vicuna server next to the plugin's `install.json`, and the server checks it every time it
starts the agent. If an upgrade changes `command`, `args` or the usage command, the harness is disabled again until the user approves
the new command. Uninstalling the plugin removes the harness, and sessions that used it show a message asking you
to pick another harness.

### What Vicuna does with an ACP agent

`initialize` → `session/new` (cwd = the session workspace, plus the user's MCP servers and Vicuna's own
`vicuna-mcp`) or `session/load` to resume → `session/set_config_option {configId: "model"}`
(falling back to `session/set_model`) → `session/prompt`, streaming `session/update`. Permission requests
are approved automatically or shown as cards, depending on the user's setting. Vicuna runs one process per
workspace, started in that folder.
