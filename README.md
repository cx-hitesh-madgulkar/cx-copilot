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

## Next steps (after the scaffold is confirmed working)

- Replace the logging commands in [hooks.json](plugins/cx-security/hooks.json) with the real
  Checkmarx policy behavior (the Copilot equivalent of `cx hooks ...`), and add a
  CLI-presence pre-check.
- Add `.mcp.json` / a `mcpServers` entry to `plugin.json` so the `cx-security-asca` skill can
  reach the Checkmarx `codeRemediation` MCP tool.
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
