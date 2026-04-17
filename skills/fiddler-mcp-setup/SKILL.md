---
name: fiddler-mcp-setup
description: >
  Set up the Fiddler Everywhere MCP server connection for AI agent tools (Claude Code,
  GitHub Copilot CLI, GitHub Copilot in VS Code, Cursor, OpenAI Codex CLI). ALWAYS use this skill when
  Fiddler MCP tools are not available, you cannot see Fiddler tools, get an authentication
  error connecting to Fiddler, need to configure Fiddler MCP for the first time, or hear
  "tool not found" errors when trying to use other Fiddler skills. Calls Fiddler's local
  API endpoint to get or generate the MCP API key automatically, writes the correct
  config file for the detected agent, gitignores it, and verifies the connection end-to-end.
  The only prerequisite is that Fiddler Everywhere is running.
---

# Fiddler MCP Setup

Configure the Fiddler Everywhere MCP server so that agent tools can call Fiddler's traffic
inspection, status, and session APIs.
---

## Operating rules

- **Shell-first.** MCP is not yet configured, so you cannot use MCP tools.
- **Sequential execution only.** Follow steps strictly in order — do not run steps or scripts in parallel. Each step may depend on values produced by the previous one.
- **Execute provided scripts directly** - do not modify or substitute the existing scripts.
- **On Windows** - Detect opened terminal. Only if it is not powershell - wrap and run the scripts with: powershell.exe -Command 'script'. Use single quotes to wrap the script!
- **`curl` only for Steps 3 through 5.** No other raw HTTP requests.
- **The MCP path is always `/mcp`.** Do not attempt to discover or vary it.
- **Direct path checks only** when detecting agent directories (e.g. `test -d .vscode`). 
Never use recursive globs (`**`) or `rg`/`grep` without a scoped directory argument.

---

## Step 1 — Verify Fiddler is installed and running

Before any MCP configuration, confirm Fiddler Everywhere is installed and the MCP
listener is reachable.

### Check installation

**macOS:**
```bash
if [ -d "/Applications/Fiddler Everywhere.app" ]; then echo "INSTALLED"; else echo "NOT_INSTALLED"; fi
```

**Linux:**
```bash
if command -v fiddler-everywhere &>/dev/null || ls ~/Downloads/FiddlerEverywhere.AppImage &>/dev/null; then echo "INSTALLED"; else echo "NOT_INSTALLED"; fi
```

**Windows (PowerShell):**
```powershell
$installed = Get-ItemProperty `
  "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*",
  "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*",
  "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" `
  -ErrorAction SilentlyContinue |
  Where-Object { $_.DisplayName -like "*Fiddler Everywhere*" }
if ($installed) { "INSTALLED" } else { "NOT_INSTALLED" }
```

If `NOT_INSTALLED`: stop and tell the user:
> "Fiddler Everywhere is not installed. Please install it first."

---

## Step 2 — Detect agent and check for existing configuration

### 2a — Detect the agent

Use a three-tier strategy. Stop at the first tier that yields exactly one match.

#### Tier 1 — Environment variables (which agent is running this shell right now)

macOS / Linux:
```bash
# VS Code / GitHub Copilot
[ -n "$VSCODE_PID" ] || [ "$TERM_PROGRAM" = "vscode" ] && echo "vscode"
# Cursor
[ -n "$CURSOR_TRACE_ID" ] || [ "$TERM_PROGRAM" = "cursor" ] && echo "cursor"
# Claude Code CLI
[ -n "$CLAUDE_CODE_ENTRYPOINT" ] && echo "claude-code"
# GitHub Copilot CLI
[ -n "$GITHUB_COPILOT_CLI" ] && echo "copilot-cli"
# OpenAI Codex CLI
[ -n "$OPENAI_CODEX" ] && echo "codex"
```

Windows (PowerShell):
```powershell
if ($env:VSCODE_PID -or $env:TERM_PROGRAM -eq "vscode") { "vscode" }
if ($env:CURSOR_TRACE_ID -or $env:TERM_PROGRAM -eq "cursor") { "cursor" }
if ($env:CLAUDE_CODE_ENTRYPOINT) { "claude-code" }
if ($env:GITHUB_COPILOT_CLI) { "copilot-cli" }
if ($env:OPENAI_CODEX) { "codex" }
```

If exactly one result is printed, set `AGENT` to that value and skip to the mapping table.
If more than one result is printed, proceed to Tier 2.
If no results, proceed to Tier 2.

> Claude Desktop does not inject env vars into child shells. If no env var matches, it may still be the active agent — check via Tier 3.

#### Tier 2 — Parent process name (OS-portable, doesn't rely on documented env vars)

macOS / Linux:
```bash
ps -p $PPID -o comm= 2>/dev/null
```

Windows (PowerShell):
```powershell
(Get-Process -Id (Get-CimInstance Win32_Process -Filter "ProcessId=$PID").ParentProcessId).Name
```

Match the output against known process names:

| Process name contains | Agent |
|-----------------------|-------|
| `code`, `code-helper` | `vscode` |
| `cursor` | `cursor` |
| `claude` | `claude-code` |
| `copilot` | `copilot-cli` |
| `codex` | `codex` |

If matched unambiguously, set `AGENT` and skip to the mapping table.
If still ambiguous or unrecognised, proceed to Tier 3.

#### Tier 3 — Filesystem markers (fallback only)

```bash
test -f ~/.copilot/mcp-config.json && echo "copilot-cli"                                              # GitHub Copilot CLI
test -d .claude && echo "claude-code"                                                                  # Claude Code CLI
test -f "$HOME/Library/Application Support/Claude/claude_desktop_config.json" && echo "claude-desktop" # Claude Desktop (macOS)
test -f "$APPDATA/Claude/claude_desktop_config.json" && echo "claude-desktop"                          # Claude Desktop (Windows)
test -d .vscode && echo "vscode"                                                                        # VS Code / GitHub Copilot
test -d .cursor && echo "cursor"                                                                        # Cursor
test -d ~/.codex && echo "codex"                                                                        # OpenAI Codex CLI
which copilot 2>/dev/null && echo "copilot-cli-in-path"                                                # Copilot CLI fallback
which codex  2>/dev/null && echo "codex-in-path"                                                       # Codex CLI fallback
```

If **multiple** markers match, do **not** guess. Ask the user:
> "Multiple agent environments were detected on this machine. Which one are you setting Fiddler MCP up for? (Claude Desktop, Claude Code CLI, GitHub Copilot CLI, VS Code / GitHub Copilot, Cursor, or OpenAI Codex CLI)"

---

#### Agent → config mapping

| Agent | Set `AGENT=` | Set `CONFIG_FILE=` |
|-------|-------------|-------------------|
| `vscode` | `vscode` | `~/Library/Application Support/Code/User/mcp.json` (macOS/Linux) or `%APPDATA%\Code\User\mcp.json` (Windows) |
| `cursor` | `cursor` | `~/.cursor/mcp.json` |
| `claude-code` | `claude-code` | `~/.claude.json` |
| `claude-desktop` | `claude-desktop` | `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows) |
| `copilot-cli` or `copilot-cli-in-path` | `copilot-cli` | `~/.copilot/mcp-config.json` |
| `codex` or `codex-in-path` | `codex` | `~/.codex/config.toml` |

If no tier yields a match, ask:
> "Which agent are you setting this up for? (Claude Desktop, Claude Code CLI, GitHub Copilot CLI, VS Code / GitHub Copilot, Cursor, or OpenAI Codex CLI)"

Use the user's answer to set `AGENT` and `CONFIG_FILE`.

### 2b — Check for existing Fiddler config

With `CONFIG_FILE` now known, check whether a Fiddler entry already exists:

```bash
grep -l "fiddler" "$CONFIG_FILE" 2>/dev/null
```

| Result | Action |
|--------|--------|
| File matched | A Fiddler config already exists. Read the file and show the user the current `url` and masked key (`xxxxxxxx…`). Ask: **"Fiddler MCP is already configured in `$CONFIG_FILE`. Do you want to re-run setup to refresh the API key, or is something not working?"** Only continue if the user confirms. |
| No match | No existing config found. Proceed to Step 3. |

---

## Step 3 — Discover the MCP port

Fiddler Everywhere listens on port `8868` by default. Before making any calls, confirm
the correct port is reachable and set `PORT` for all subsequent steps.

macOS / Linux:
```bash
curl -s -o /dev/null -w "%{http_code}" -X POST "http://localhost:8868/mcp" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":0,"method":"ping","params":{}}'
```

Windows (PowerShell):
```powershell
curl.exe -s -o NUL -w "%{http_code}" -X POST "http://localhost:8868/mcp" `
  -H "Content-Type: application/json" `
  -H "Accept: application/json, text/event-stream" `
  -d '{\"jsonrpc\":\"2.0\",\"id\":0,\"method\":\"ping\",\"params\":{}}'
```

| Result | Action |
|--------|--------|
| Any response (even `4xx`) | Port `8868` is reachable. Set `PORT=8868` and proceed to Step 4. |
| `000` (connection refused) | Port `8868` is not listening. Run the discovery script below. |

**Port discovery script:**

macOS / Linux:
```bash
python3 -c "
import json, glob, os
ports = set()
for f in glob.glob(os.path.expanduser('~/.fiddler/*/Settings/appsettings.json')):
    try:
        p = json.load(open(f)).get('MCPServerSettings', {}).get('Port')
        if p and p != 8868:
            ports.add(p)
    except: pass
print(' '.join(str(p) for p in ports) if ports else 'none')
"
```

Windows (PowerShell):
```powershell
$ports = Get-ChildItem "$env:USERPROFILE\.fiddler\*\Settings\appsettings.json" -ErrorAction SilentlyContinue |
  ForEach-Object { (Get-Content $_ | ConvertFrom-Json).MCPServerSettings.Port } |
  Where-Object { $_ -and $_ -ne 8868 } | Select-Object -Unique
Write-Output $(if ($ports) { $ports -join ' ' } else { 'none' })
```

Try a `ping` call to each discovered port. Set `PORT` to the first port that responds.
If no port responds, Fiddler is not running. Launch it using the commands below, wait 15 seconds, then retry discovery from the top of Step 3.

**Launch Fiddler:**

macOS:
```bash
open -a "Fiddler Everywhere" && sleep 15
```

Linux:
```bash
(nohup fiddler-everywhere &>/dev/null &); sleep 15
```

Windows:
**Important** On windows if the current terminal used is bash, use the specific **GitBash** script.

**PowerShell**
```powershell
$candidates = @(
  "$env:LOCALAPPDATA\Programs\Fiddler Everywhere\Fiddler Everywhere.exe",
  "C:\Program Files\Fiddler Everywhere\Fiddler Everywhere.exe",
  "C:\Program Files (x86)\Fiddler Everywhere\Fiddler Everywhere.exe"
)
$fiddlerExe = $candidates | Where-Object { Test-Path $_ } | Select-Object -First 1
if ($fiddlerExe) {
  $cmdLine = '"' + $fiddlerExe + '"'
  Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{ CommandLine = $cmdLine } | Out-Null
  Start-Sleep 15
}
```

**Git Bash**
```bash
FIDDLER_EXE=""
for dir in "$LOCALAPPDATA/Programs/Fiddler Everywhere" \
           "/c/Program Files/Fiddler Everywhere" \
           "/c/Program Files (x86)/Fiddler Everywhere"; do
  if [ -f "$dir/Fiddler Everywhere.exe" ]; then
    FIDDLER_EXE="$dir/Fiddler Everywhere.exe"
    break
  fi
done
if [ -n "$FIDDLER_EXE" ]; then
  "$FIDDLER_EXE" &
  sleep 15
fi
```

**Important:** Wait 15 seconds for Fiddler to launch, before continuing with the next steps!

If still unreachable after relaunch:
> "Fiddler Everywhere is not reachable. Please verify it is running and try again."

Once `PORT` is established, use it for all subsequent steps.

> `PORT` is used in all commands from Step 4 onwards.

---

## Step 4 — Verify login and get the API key

### 4a — Get or generate the API key

Once logged in, call the key-management endpoint on `PORT`.
It returns the existing API key (or generates one automatically if none exists) along with the MCP URL.

```bash
curl -s -X POST "http://localhost:$PORT/api/McpManagement/GetOrGenerateApiKey"
```

Expected success response:

```json
{
  "apiKey": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "port": 8868,
  "url": "http://localhost:8868/mcp"
}
```

Extract `apiKey` as **KEY** and `url` as **MCP_URL**.

| Result | Meaning | Action |
|--------|---------|--------|
| JSON with `apiKey` field | Success | Extract `apiKey` as KEY and `url` as MCP_URL. Proceed to Step 5. |
| `403` / access-denied body | Subscription plan does not include MCP | Stop: "Your Fiddler plan does not include MCP access. Please upgrade your subscription." |
| User is not logged in | User not logged in yet. | Step 4b - Initiate Login |
| Any other failure | Unexpected failure | Note the response and ask the user to check Fiddler is running correctly. |

### 4b - Initiate Login (if user is not logged in):

The steps are: `initialize` -> `list_tools` -> `initiate_login`
Send an MCP `initialize` request and get the returned session id.

macOS / Linux:
```bash
SESSION_ID=$(curl -si -X POST "http://localhost:$PORT/mcp" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"mcp-setup","version":"1.0"}}}' \
  | grep -i "^mcp-session-id:" | awk '{print $2}' | tr -d '\r')
echo "Session ID: $SESSION_ID"
```

Windows (PowerShell):
```powershell
$initResp = curl.exe -si -X POST "http://localhost:$PORT/mcp" `
  -H "Content-Type: application/json" `
  -H "Accept: application/json, text/event-stream" `
  -d '{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"initialize\",\"params\":{\"protocolVersion\":\"2024-11-05\",\"capabilities\":{},\"clientInfo\":{\"name\":\"mcp-setup\",\"version\":\"1.0\"}}}'
$SESSION_ID = ($initResp | Select-String "(?i)mcp-session-id:\s*(\S+)").Matches.Groups[1].Value.Trim()
Write-Host "Session ID: $SESSION_ID"
```

If `$SESSION_ID` is empty, the server did not return a session ID — Fiddler may not be
fully started. Wait a few seconds and retry.

Then fetch the tools list using the same session.

macOS / Linux:
```bash
curl -s -X POST "http://localhost:$PORT/mcp" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
```

Windows (PowerShell):
```powershell
curl.exe -s -X POST "http://localhost:$PORT/mcp" `
  -H "Content-Type: application/json" `
  -H "Accept: application/json, text/event-stream" `
  -H "Mcp-Session-Id: $SESSION_ID" `
  -d '{\"jsonrpc\":\"2.0\",\"id\":2,\"method\":\"tools/list\",\"params\":{}}'
```

Confirm `initiate_login` appears in the response before proceeding.

Now call `initiate_login` using the same session.

macOS / Linux:
```bash
curl -s -X POST "http://localhost:$PORT/mcp" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"initiate_login","arguments":{}}}'
```

Windows (PowerShell):
```powershell
curl.exe -s -X POST "http://localhost:$PORT/mcp" `
  -H "Content-Type: application/json" `
  -H "Accept: application/json, text/event-stream" `
  -H "Mcp-Session-Id: $SESSION_ID" `
  -d '{\"jsonrpc\":\"2.0\",\"id\":4,\"method\":\"tools/call\",\"params\":{\"name\":\"initiate_login\",\"arguments\":{}}}'
```

This opens a Chrome window for authentication. Tell the user:
> "A login window has been opened. Please complete sign-in, then let me know when done."

When done - retry Step 4a.

---

## Step 5 — Probe the server

Verify the key is valid using the MCP_URL from Step 3.

macOS / Linux:
```bash
curl -s -o /dev/null -w "%{http_code}" -X POST "MCP_URL" \
  -H "Authorization: ApiKey KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"mcp-setup","version":"1.0"}}}'
```

Windows (PowerShell):
```powershell
curl.exe -s -o NUL -w "%{http_code}" -X POST "MCP_URL" `
  -H "Authorization: ApiKey KEY" `
  -H "Content-Type: application/json" `
  -H "Accept: application/json, text/event-stream" `
  -d '{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"initialize\",\"params\":{\"protocolVersion\":\"2024-11-05\",\"capabilities\":{},\"clientInfo\":{\"name\":\"mcp-setup\",\"version\":\"1.0\"}}}'
```

| Code | Meaning | Action |
|------|---------|--------|
| `200` or any 2xx | Key valid (response may be an SSE stream — that is normal) | Proceed to Step 6. |
| `000` | Fiddler stopped between Step 4 and now | Re-run from Step 3. |
| `401` | Key rejected | Re-run Step 4 to regenerate the key, then retry. |
| `403` | Subscription plan does not include MCP | Stop: "Your Fiddler plan does not include MCP access." |
| Anything else | Unexpected — note the code | Proceed to Step 6 anyway. |

**Do not** retry with any path other than `/mcp`. The Fiddler MCP path is always `/mcp`.

---

## Step 6 — Write the config file

Use `CONFIG_FILE` set in Step 2. 
Use the exact KEY and MCP_URL values from Step 4.

**If `CONFIG_FILE` already exists:** read it first, then add only the `fiddler`
server block. Do **not** remove or overwrite other existing server entries.

**If it does not exist:** create it using the template for `AGENT` below.

### `AGENT=copilot-cli`

```json
{
  "mcpServers": {
    "fiddler": {
      "type": "http",
      "url": "MCP_URL",
      "headers": {
        "Authorization": "ApiKey KEY"
      },
      "tools": ["*"]
    }
  }
}
```

> Note: `"tools": ["*"]` is required by Copilot CLI — omitting it disables all tools.

---

### `AGENT=claude-desktop`

Claude Desktop does not support the `http` MCP transport directly. Use `npx mcp-remote` as a bridge.

```json
{
  "mcpServers": {
    "fiddler": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "MCP_URL",
        "--header",
        "Authorization:ApiKey KEY"
      ]
    }
  }
}
```

> Note: `npx` must be available on the system PATH. If Node.js is not installed, direct the user to https://nodejs.org.

---

### `AGENT=claude-code`

```json
{
  "mcpServers": {
    "fiddler": {
      "type": "http",
      "url": "MCP_URL",
      "headers": {
        "Authorization": "ApiKey KEY"
      }
    }
  }
}
```

### `AGENT=vscode`

```json
{
  "servers": {
    "fiddler": {
      "type": "http",
      "url": "MCP_URL",
      "headers": {
        "Authorization": "ApiKey KEY"
      }
    }
  }
}
```

### `AGENT=cursor`

```json
{
  "mcpServers": {
    "fiddler": {
      "url": "MCP_URL",
      "headers": {
        "Authorization": "ApiKey KEY"
      }
    }
  }
}
```

### `AGENT=codex`

Codex CLI uses TOML.

```toml
[mcp_servers.fiddler]
enabled = true
url = "MCP_URL"

[mcp_servers.fiddler.http_headers]
Authorization = "ApiKey KEY"
```

---

### Git safety

**All agents** use global config files outside any repository. Inform the user:
> "Your Fiddler API key is stored in `CONFIG_FILE`. Keep this file private."