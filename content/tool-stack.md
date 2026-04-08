---
title: Tool Stack
lang: en
tags:
  - tools
draft: "false"
---

A living document of my tool stack.

## Devices

**Macbook Pro (M1)**

**Mac Mini (M2)**

## Environment

**[OrbStack](https://orbstack.dev/)**

* Running Linux containers and VMs on macOS
* Faster & more lightweight than Docker Desktop

**[uv](https://docs.astral.sh/uv/)**

* Blazing fast Python project manager (cargo/npm for Python)
* Replacing `pip`, `conda` or `pyenv`

## Editing

**[Zed](https://zed.dev/)**

* Minimal, high-performance code editor
* Fast startup
* Comfortable remote development
* Seamless collaboration through panel sharing
* Native AI integration (Copilot, Claude, …) through [ACP](https://agentclientprotocol.com/get-started/introduction)

**[Obsidian](https://obsidian.md/)**

* Markdown-based knowledge management
* Used for tech notes, research note, and personal documentation
* Publish markdown notes to [my personal website](https://allenyolk.github.io/) through **[Quartz](https://quartz.jzhao.xyz/)**

## Terminal

**[Ghostty](https://ghostty.org/)**

* GPU-accelerated terminal emulator
* Modern features & smooth rendering

**[Zsh](https://www.zsh.org/) + [Oh My Zsh](https://ohmyz.sh/)**

* Customizable shell with plugins and themes
* Productivity-focused enhancements
* Sync dotfiles through Github

## AI

**[GitHub Copilot](https://github.com/features/copilot)**

* AI code completion and next-edit suggestions
* Integrates directly into Zed, VS Code and terminals
* For simple/short-term coding tasks

**[Claude Code](https://claude.ai/)**

* Advanced AI agent for code understanding and generation
* Integrates with Zed, VS Code and terminals
* For complex refactoring, code review, and documentation

**[Hermes Agent](https://hermes-agent.nousresearch.com/)**

* Autonomous AI agent with local system access
* Integrates with Claude and Copilot APIs via a local gateway
* Capable of file manipulation, terminal execution, and task planning

**[LibreChat](https://github.com/danny-avila/LibreChat)**

* Open-source, self-hosted LLM chat UI & service
* Supports multiple AI providers (OpenAI, Anthropic, etc.)
* Also acts as the UI for Hermes Agent

## Network

**[Tailscale](https://tailscale.com/)**

* Zero-config VPN for secure, private networking across devices
* **Tailscale + [macOS's Screen Sharing](https://support.apple.com/en-sg/guide/mac-help/mh14066/mac)**: manipulate my Mac Mini from my Macbook
* **Tailscale + LibreChat**: chat with LLMs/agents on my cell phone

> [!summary]
>
> ```mermaid
> %%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#0d1117', 'edgeLabelBackground':'#161b22', 'tertiaryColor': '#161b22', 'primaryTextColor': '#c9d1d9', 'lineColor': '#58a6ff', 'fontSize': '16px','fontFamily': 'Inter, system-ui, sans-serif'}}}%%
> 
> C4Context
>     Boundary(secure_network, "Tailscale Private Network", "") {
>         Person(user, "User (at home)", "")
>         System(iphone, "iPhone", "")
>         System(macbook, "Macbook", "")
>
>         Boundary(mac_mini_box, "Mac Mini", "") {
>             System(librechat, "LibreChat UI", "")
>             System(zed, "Zed Editor", "")
>             System(hermes_gateway, "Hermes Gateway", "")
>             System(orbstack, "OrbStack", "")
>             System(macos, "macOS on Mac Mini", "")
>         }
>     }
>
>     Boundary(providers, "LLM Providers", "") {
>         System_Ext(claude_api, "Claude API", "")
>         System_Ext(copilot_api, "GitHub Copilot API", "")
>     }
>
> 	Rel(user, iphone, "")
> 	Rel(user, macbook, "")
>     Rel(iphone, librechat, "")
>     Rel(macbook, librechat, "")
>     Rel(macbook, macos, "")
>     Rel(librechat, orbstack, "")
>     Rel(librechat, claude_api, "")
>     Rel(librechat, hermes_gateway, "")
>     Rel(zed, claude_api, "")
>     Rel(zed, copilot_api, "")
>     Rel(hermes_gateway, claude_api, "")
>     Rel(hermes_gateway, copilot_api, "")
>     Rel(hermes_gateway, macos, "")
>
>     UpdateElementStyle(user, $bgColor="#ff7b7222", $borderColor="#f85149") 
>     UpdateElementStyle(iphone, $bgColor="#ff7b7211", $borderColor="#f85149") 
>     UpdateElementStyle(macbook, $bgColor="#ff7b7211", $borderColor="#f85149") 
>     UpdateElementStyle(librechat, $bgColor="#58a6ff22", $borderColor="#1f6feb") 
>     UpdateElementStyle(hermes_gateway, $bgColor="#3fb95022", $borderColor="#238636") 
>     UpdateElementStyle(orbstack, $bgColor="#3fb95011", $borderColor="#238636") 
>     UpdateElementStyle(claude_api, $bgColor="#d2992222", $borderColor="#9e6a03") 
>     UpdateElementStyle(copilot_api, $bgColor="#d2992222", $borderColor="#9e6a03") 
>     UpdateElementStyle(macos, $bgColor="#8b949e22", $borderColor="#484f58") 
>     UpdateElementStyle(zed, $bgColor="#8b949e11", $borderColor="#484f58") 
>
>     UpdateRelStyle(macbook, macos, $lineColor="#9e6a03", $textColor="#9e6a03") 
>     UpdateRelStyle(librechat, orbstack, $lineColor="#1f6feb", $textColor="#1f6feb") 
>     UpdateRelStyle(hermes_gateway, macos, $lineColor="#238636", $textColor="#238636")
> ```
