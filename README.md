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

## Prerequisites

If you haven't set up Fiddler with your agent yet, start with the [`fiddler-mcp-setup`](skills/fiddler-mcp-setup/SKILL.md) skill or [`fiddler-download-setup`](skills/fiddler-download-setup/SKILL.md) if you don't have Fiddler installed at all.