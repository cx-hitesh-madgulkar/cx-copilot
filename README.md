# Checkmarx Copilot Marketplace

A plugin marketplace for **GitHub Copilot CLI**, mirroring the objective of
[`cxone-scanners`](../cxone-scanners) (the Claude Code marketplace): bring Checkmarx CxOne
security capabilities — install/setup and ASCA scan + remediation skills — into the agent.

> **Status: test scaffold.** The hooks in this repo are intentionally **logging-only
> placeholders**. They do not enforce any security policy yet. Their only job right now is to
> prove the plugin loads and that each hook event fires in Copilot CLI. Once that is confirmed,
> the hook commands will be replaced with the real `cx hooks ...` policy behavior.

## Repository structure

```
Coplitot-Marketplace/
├── .github/
│   └── plugin/
│       └── marketplace.json          # Marketplace manifest (Copilot-native location)
├── plugins/
│   └── cx-security/
│       ├── plugin.json               # Plugin manifest (at plugin root)
│       ├── hooks.json                # Logging-only test hooks
│       ├── .mcp.json                 # Checkmarx MCP server (HTTP) — placeholders only
│       └── skills/
│           ├── cx-cli-setup/
│           │   └── SKILL.md          # Copied verbatim from cxone-scanners
│           └── cx-security-asca/
│               └── SKILL.md          # Copied verbatim from cxone-scanners
└── README.md
```

### How this differs from the Claude Code (`cxone-scanners`) layout

| Concern | Claude Code | Copilot CLI (this repo) |
|---|---|---|
| Plugin manifest location | `.claude-plugin/plugin.json` | `plugin.json` at plugin root |
| Marketplace manifest | `.claude-plugin/marketplace.json` | `.github/plugin/marketplace.json` |
| Hooks schema | `{ "hooks": { "PreToolUse": [{ "matcher", "hooks": [...] }] } }` | `{ "version": 1, "hooks": { "preToolUse": [{ "matcher", "type", "bash"/"powershell" }] } }` |
| Hook events | `PreToolUse`, `UserPromptSubmit`, `Stop`, … | `preToolUse`, `userPromptSubmitted`, `agentStop`, `sessionStart`, … |
| Plugin root variable | `${CLAUDE_PLUGIN_ROOT}` | *(no documented equivalent — these hooks use inline commands instead)* |
| Cross-platform commands | single `command` | separate `bash` + `powershell` keys |

## What the test hooks do

Every wired event appends one line to **`$HOME/CxCopilot/info.log`** (on Windows that is
`%USERPROFILE%\CxCopilot\info.log`). The directory is created on first run.

| Event | Logged line |
|---|---|
| `sessionStart` | `… [sessionStart] plugin cx-security loaded` — confirms the plugin loaded |
| `userPromptSubmitted` | `… [userPromptSubmitted] <payload>` — the submitted prompt payload |
| `preToolUse` (matcher `.*`) | `… [preToolUse] <payload>` — fires on **every** tool call; the payload's `tool_input` contains the **file name** for write/edit tools |
| `agentStop` | `… [agentStop] session ended` |

The hooks read the JSON payload Copilot pipes on stdin and log it whole, so the file name (and
all other tool input) is captured without needing to parse JSON in-shell. Each hook is
cross-platform: `bash` runs on Linux/macOS, `powershell` runs on Windows.

## Testing in GitHub Copilot CLI

1. **Push this repo** (the `Coplitot-Marketplace` folder as the repo root) to your personal
   GitHub account, e.g. `your-username/cx-copilot-marketplace`.

2. **Register the marketplace** from the repo:

   ```shell
   copilot plugin marketplace add your-username/cx-copilot-marketplace
   ```

3. **Install the plugin**:

   ```shell
   copilot plugin install cx-security
   ```

   For purely local testing (no push) you can instead point Copilot at the plugin directory:

   ```shell
   copilot plugin install ./plugins/cx-security
   ```

4. **Trigger activity** — start a Copilot CLI session in any project, submit a prompt, and let
   it run a tool (e.g. write or edit a file).

5. **Verify the hooks fired** — check the log:

   ```shell
   # macOS / Linux
   cat "$HOME/CxCopilot/info.log"

   # Windows (PowerShell)
   Get-Content "$HOME\CxCopilot\info.log"
   ```

   You should see a `sessionStart` line, a `userPromptSubmitted` line, one `preToolUse` line per
   tool call (with the file name inside the payload), and an `agentStop` line when the session ends.

## Checkmarx MCP server (`.mcp.json`)

[.mcp.json](plugins/cx-security/.mcp.json) declares the Checkmarx MCP server so the
`cx-security-asca` skill can reach the `mcp__Checkmarx__codeRemediation` tool. It uses Copilot's
remote **HTTP** transport:

```json
{
  "mcpServers": {
    "Checkmarx": {
      "type": "http",
      "url": "<Cx_MCP_URL>",
      "headers": {
        "cx-origin": "VsCode",
        "Authorization": "<AUTH_TOKEN>"
      },
      "tools": ["*"]
    }
  }
}
```

The server is named **`Checkmarx`** on purpose — that name produces the `mcp__Checkmarx__*` tool
prefix the skill expects. `"tools": ["*"]` exposes all of the server's tools.

> ⚠️ **Do not commit a real `Authorization` token.** Copilot CLI does **not** support environment
> variable expansion in `.mcp.json`, and this file is distributed with the plugin — a real token
> pushed to your repo is a leaked credential. Keep the committed file as placeholders.

**Two ways to supply real credentials:**

- **Recommended — user-scope config (secret never enters the repo):** add the same server block to
  your machine's `~/.copilot/mcp-config.json` with the real `url` and token. The skill's tools
  become available in every session and nothing sensitive is committed. In that case the plugin's
  `.mcp.json` stays as documentation/placeholders.
- **Local-only edit:** fill `<Cx_MCP_URL>` and `<AUTH_TOKEN>` in the plugin's `.mcp.json` for
  testing, but **never push** that version (e.g. keep it on an untracked local copy). `Authorization`
  is usually `Bearer <token>`. Adjust `cx-origin` if your tenant expects a different origin value.

## Next steps (after the scaffold is confirmed working)

- Replace the logging commands in [hooks.json](plugins/cx-security/hooks.json) with the real
  Checkmarx policy behavior (the Copilot equivalent of `cx hooks ...`), and add a
  CLI-presence pre-check.
- Verify the MCP connection: with real credentials configured, run the `cx-security-asca` skill and
  confirm `mcp__Checkmarx__codeRemediation` is reachable.
- Tighten the `preToolUse` matcher from `.*` to the specific Copilot tool names once their
  exact identifiers are confirmed.

## References

- [Creating a plugin for GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-creating)
- [Creating a plugin marketplace for GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-marketplace)
- [Using hooks with GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/use-hooks)
- [GitHub Copilot hooks reference](https://docs.github.com/en/copilot/reference/hooks-reference)
- [GitHub Copilot CLI plugin reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-plugin-reference)

## License

MIT
