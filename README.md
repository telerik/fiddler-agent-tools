# Fiddler Everywhere Agent Skills

This repository contains **official agent skills built exclusively for [Fiddler Everywhere](https://www.telerik.com/fiddler/fiddler-everywhere)** — the cross-platform web debugging proxy by Progress Telerik. All skills in this repo require Fiddler Everywhere to be installed and running on the user's machine; they are not compatible with Fiddler Classic or any other proxy tool.

These skills extend agents (GitHub Copilot, Claude Code, Cursor, Codex, etc.) with the ability to directly interact with Fiddler Everywhere through its **MCP (Model Context Protocol) server**. Once connected, the agent can capture, inspect, and analyze live HTTP/HTTPS traffic without the user having to leave their editor. 

Use these skills any time you want your AI coding agent to reason about real network traffic rather than relying on guesses or static code analysis alone.

## Installation

### Quick install (via [skills.sh](https://skills.sh))

The fastest way to install all Fiddler skills is through the `skills` CLI — no manual file placement needed:

```bash
npx skills add telerik/fiddler-agent-tools/skills
```

This automatically detects your agent and places the skill files in the correct directory.

To install a single skill:

```bash
npx skills add telerik/fiddler-agent-tools --skill fiddler-mcp-setup
npx skills add telerik/fiddler-agent-tools --skill fiddler-download-setup
npx skills add telerik/fiddler-agent-tools --skill fiddler-traffic-debugging
```

### Manual setup

If you prefer to install skills manually, clone this repository and copy the skill folders to the appropriate directory for your agent.

```bash
git clone https://github.com/telerik/fiddler-agent-tools.git
```

Then copy the contents of the `skills/` folder to the matching path:

| Agent | Skills directory |
|-------|-----------------|
| **GitHub Copilot (VS Code)** | `~/.copilot/skills/` (macOS/Linux) · `%USERPROFILE%\.copilot\skills\` (Windows) |
| **Cursor** | `~/.cursor/skills/` (macOS/Linux) · `%USERPROFILE%\.cursor\skills\` (Windows) |
| **Claude Code** | `~/.claude/skills/` |
| **Claude Desktop** | `~/.claude/skills/` |
| **GitHub Copilot CLI** | `~/.copilot/skills/` |
| **OpenAI Codex CLI** | `~/.codex/skills/` |

For example, on macOS/Linux for GitHub Copilot in VS Code:

```bash
mkdir -p ~/.copilot/skills
cp -r skills/* ~/.copilot/skills/
```

Each skill folder must contain its `SKILL.md` file. After copying, your agent will automatically detect and use the skills when a matching request is made.

## What are skills?

Skills are instruction files that teach coding agents (GitHub Copilot, Claude Code, Cursor, Codex, VsCode, etc.) how to perform specific tasks. Each skill lives in its own folder as a `SKILL.md` file and is loaded by the agent when the user's request matches the skill's description.

## Skills

| Skill | Description |
|-------|-------------|
| [`fiddler-download-setup`](skills/fiddler-download-setup/SKILL.md) | Download, install, and launch Fiddler Everywhere from scratch. |
| [`fiddler-mcp-setup`](skills/fiddler-mcp-setup/SKILL.md) | Connect your agent to the Fiddler Everywhere MCP server. Use this when Fiddler tools aren't available in your agent session. |
| [`fiddler-traffic-debugging`](skills/fiddler-traffic-debugging/SKILL.md) | Verify that a feature's HTTP calls completed correctly by analyzing captured traffic, grouped by endpoint. |

## Prerequisites

If you haven't set up Fiddler with your agent yet, start with the [`fiddler-mcp-setup`](skills/fiddler-mcp-setup/SKILL.md) skill or [`fiddler-download-setup`](skills/fiddler-download-setup/SKILL.md) if you don't have Fiddler installed at all.