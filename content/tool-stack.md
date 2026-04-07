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

**[LibreChat](https://github.com/danny-avila/LibreChat)**

* Open-source, self-hosted chat UI for LLMs
* Supports multiple AI providers (OpenAI, Anthropic, etc.)

## Network

**[Tailscale](https://tailscale.com/)**

* Zero-config VPN for secure, private networking across devices
* **Tailscale + [macOS's Screen Sharing](https://support.apple.com/en-sg/guide/mac-help/mh14066/mac)**: manipulate my Mac Mini from my Macbook
* **Tailscale + LibreChat**: chat with LLMs on my cell phone
