# Fiddler Everywhere Agent Skills

This repository contains **official agent skills built exclusively for [Fiddler Everywhere](https://www.telerik.com/fiddler/fiddler-everywhere)** — the cross-platform web debugging proxy by Progress Telerik. All skills in this repo require Fiddler Everywhere to be installed and running on the user's machine; they are not compatible with Fiddler Classic or any other proxy tool.

These skills extend agents (GitHub Copilot, Claude Code, Cursor, Codex, etc.) with the ability to directly interact with Fiddler Everywhere through its **MCP (Model Context Protocol) server**. Once connected, the agent can capture, inspect, and analyze live HTTP/HTTPS traffic without the user having to leave their editor. 

Use these skills any time you want your AI coding agent to reason about real network traffic rather than relying on guesses or static code analysis alone.


## What are skills?

Skills are instruction files that teach coding agents (GitHub Copilot, Claude Code, Cursor, Codex, VsCode, etc.) how to perform specific tasks. Each skill lives in its own folder as a `SKILL.md` file and is loaded by the agent when the user's request matches the skill's description.

## Skills

| Skill | Description |
|-------|-------------|
| [`fiddler-download-setup`](skills/fiddler-download-setup/SKILL.md) | Download, install, and launch Fiddler Everywhere from scratch. |
| [`fiddler-mcp-setup`](skills/fiddler-mcp-setup/SKILL.md) | Connect your agent to the Fiddler Everywhere MCP server. Use this when Fiddler tools aren't available in your agent session. |
| [`fiddler-feature-verification`](skills/fiddler-feature-verification/SKILL.md) | Verify that a feature's HTTP calls completed correctly by analyzing captured traffic, grouped by endpoint. |

## Installation

Clone this repository and copy the skill folders into your agent's skills directory.

### Claude Code

```bash
# Clone the repo
git clone https://github.com/nickolay-aspect/fiddler-agent-tools.git

# Copy all skills into the Claude Code skills directory
cp -r fiddler-agent-tools/skills/* ~/.claude/skills/
```

On Windows (PowerShell):
```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills\" | Out-Null
Copy-Item -Recurse fiddler-agent-tools\skills\* "$env:USERPROFILE\.claude\skills\"
```

### GitHub Copilot (VS Code)

Copy the skill folders into your VS Code user prompts directory:

```bash
cp -r fiddler-agent-tools/skills/* ~/.vscode/prompts/skills/
```

On Windows (PowerShell):
```powershell
Copy-Item -Recurse fiddler-agent-tools\skills\* "$env:APPDATA\Code\User\prompts\skills\"
```

### Cursor

Copy the skill folders into the Cursor skills directory:

```bash
cp -r fiddler-agent-tools/skills/* ~/.cursor/skills/
```

On Windows (PowerShell):
```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.cursor\skills\" | Out-Null
Copy-Item -Recurse fiddler-agent-tools\skills\* "$env:USERPROFILE\.cursor\skills\"
```

> **Note:** After copying, restart your agent or editor session so the new skills are detected.

## Prerequisites

If you haven't set up Fiddler with your agent yet, start with the [`fiddler-mcp-setup`](skills/fiddler-mcp-setup/SKILL.md) skill or [`fiddler-download-setup`](skills/fiddler-download-setup/SKILL.md) if you don't have Fiddler installed at all.