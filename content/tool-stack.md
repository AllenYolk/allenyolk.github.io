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

**[Apple Container](https://github.com/apple/container)**

* Native macOS container runtime (VM-per-container, kata-based kernel)
* Replacement for OrbStack / Docker Desktop

**[uv](https://docs.astral.sh/uv/)**

* Blazing fast Python project manager (cargo/npm for Python)
* Replacing `pip`, `conda` or `pyenv`

## Editing

**[Zed](https://zed.dev/)**

* Minimal, high-performance code editor
* Fast startup
* Remote development
* Seamless collaboration through panel sharing
* Native AI integration (Copilot, OpenAI Codex, …) through [ACP](https://agentclientprotocol.com/get-started/introduction)

**[Obsidian](https://obsidian.md/)**

* Markdown-based knowledge management
* Used for tech notes, research note, and personal documentation
* Publish markdown notes to [my personal website](https://allenyolk.github.io/) through **[Quartz](https://quartz.jzhao.xyz/)**

**[tectonic](https://tectonic-typesetting.github.io/)**

* Lean, self-contained LaTeX engine — replaces `pdflatex` + `bibtex` + `latexmk`
* Auto-fetches LaTeX packages on demand (no TeX Live install)

## Terminal

**[Ghostty](https://ghostty.org/)**

* GPU-accelerated terminal emulator
* Modern features & smooth rendering

**[herdr](https://github.com/ogulcancelik/herdr)**

* Terminal multiplexer (tmux successor)
* Run CLI agents in the background
* Persistent sessions for monitoring remote server state 

**[Zsh](https://www.zsh.org/) + [Oh My Zsh](https://ohmyz.sh/)**

* Customizable shell with plugins and themes
* Productivity-focused enhancements
* Sync dotfiles through Github

## AI

**[Codex](https://openai.com/codex/)**

* Advanced AI agent for code understanding and generation via CLI
* Also features a Desktop GUI app for visual codebase exploration and chat
* For complex refactoring, code review, and documentation
* Integrates with Zed, VS Code and terminals
* **Remote control from phone:** drive the Codex session on my Mac mini through the [ChatGPT](https://chatgpt.com/) mobile app — no [Tailscale](https://tailscale.com/) required (the link rides on OpenAI's backend, not my LAN)

**[Hermes Agent](https://hermes-agent.nousresearch.com/)**

* Autonomous AI agent with local system access
* Primary backend: [MiniMax](https://api.minimaxi.com/) (MiniMax-M3)
* Secondary backend: [OpenCode Go](https://opencode.com/)
* Fallback: [Open Router](https://openrouter.ai/)
* Capable of file manipulation, terminal execution, and task planning
* Memory enhanced by [Holographic](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers#holographic)
* Connect to [Feishu](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/feishu)

## Network

**[Tailscale](https://tailscale.com/)**

* Zero-config VPN for secure, private networking across devices
* **Tailscale + [macOS's Screen Sharing](https://support.apple.com/en-sg/guide/mac-help/mh14066/mac)**: manipulate my Mac Mini from my Macbook
* **Tailscale + SSH** to my Mac Mini from my Macbook (after [enabling remote login](https://osxdaily.com/2022/07/08/turn-on-ssh-mac/))
	* With Zed Remote: no need to clone the repos to my Macbook
* **Tailscale + [Termius](https://termius.com/)**: SSH to my Mac Mini from my iPhone

## Summary

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#0d1117', 'edgeLabelBackground':'#161b22', 'tertiaryColor': '#161b22', 'primaryTextColor': '#c9d1d9', 'lineColor': '#58a6ff', 'fontSize': '16px','fontFamily': 'Inter, system-ui, sans-serif'}}}%%

C4Context
    Boundary(secure_network, "Tailscale Private Network", "") {
        Person(user, "User (at home)", "")
        System(iphone, "iPhone", "")
        System(macbook, "Macbook", "")

         Boundary(mac_mini_box, "Mac Mini", "") {
 	        System(zed, "Zed Editor", "")
             System(hermes_gateway, "Hermes Gateway", "")
             System(codex, "Codex", "")
             System(macos, "macOS on Mac Mini", "")
         }
    }
     Boundary(providers, "LLM Providers", "") {
	    System_Ext(openai_api, "OpenAI API", "")
	    System_Ext(copilot_api, "GitHub Copilot API", "")
	    System_Ext(minimax_api, "MiniMax API", "")
	    System_Ext(open_router, "Open Router", "")
	    System_Ext(opencode_go, "OpenCode Go", "")
	    System_Ext(chatgpt_app, "ChatGPT Mobile App", "")
	 }
	
	 Rel(user, iphone, "")
	 Rel(user, macbook, "")
	 Rel(macbook, macos, "Screen Sharing / SSH")
	 Rel(iphone, macos, "Termius + SSH")
	 Rel(macbook, hermes_gateway, "Feishu")
	 Rel(iphone, hermes_gateway, "Feishu")
	 Rel(macbook, zed, "")
	 Rel(zed, openai_api, "")
	 Rel(zed, copilot_api, "")
	 Rel(hermes_gateway, openai_api, "")
	 Rel(hermes_gateway, minimax_api, "")
	 Rel(hermes_gateway, copilot_api, "")
	 Rel(hermes_gateway, open_router, "")
Rel(hermes_gateway, opencode_go, "")
Rel(hermes_gateway, macos, "Manipulate")
Rel(iphone, chatgpt_app, "ChatGPT mobile")
Rel(chatgpt_app, codex, "remote session")
Rel(codex, openai_api, "via OpenAI backend")
%% User 红色系 %% 
	 UpdateElementStyle(user, $bgColor="#ff7b7222", $borderColor="#f85149") 
     UpdateElementStyle(iphone, $bgColor="#ff7b7211", $borderColor="#f85149") 
     UpdateElementStyle(macbook, $bgColor="#ff7b7211", $borderColor="#f85149") 
     UpdateElementStyle(chatgpt_app, $bgColor="#ff7b7211", $borderColor="#f85149") 
     %% Agent/Infrastructure 绿色系 %% 
     UpdateElementStyle(hermes_gateway, $bgColor="#3fb95022", $borderColor="#238636") 
     UpdateElementStyle(orbstack, $bgColor="#3fb95011", $borderColor="#238636") 
     %% API 金色系 %% 
     UpdateElementStyle(openai_api, $bgColor="#d2992222", $borderColor="#9e6a03") 
     UpdateElementStyle(minimax_api, $bgColor="#d2992222", $borderColor="#9e6a03")
     UpdateElementStyle(copilot_api, $bgColor="#d2992222", $borderColor="#9e6a03") 
     UpdateElementStyle(open_router, $bgColor="#d2992222", $borderColor="#9e6a03")
     UpdateElementStyle(opencode_go, $bgColor="#d2992222", $borderColor="#9e6a03")
     %% Host 灰色系 %% 
    UpdateElementStyle(macos, $bgColor="#8b949e22", $borderColor="#484f58") 
    UpdateElementStyle(zed, $bgColor="#8b949e11", $borderColor="#484f58") 
     UpdateElementStyle(codex, $bgColor="#8b949e11", $borderColor="#484f58") 
    %% --- 连线颜色微调 --- %% 
    UpdateRelStyle(macbook, macos, $lineColor="#9e6a03", $textColor="#9e6a03") 
    UpdateRelStyle(iphone, macos, $lineColor="#9e6a03", $textColor="#9e6a03")
    UpdateRelStyle(hermes_gateway, macos, $lineColor="#238636", $textColor="#238636")
    UpdateRelStyle(macbook, hermes_gateway, $lineColor="#238636", $textColor="#238636")
    UpdateRelStyle(iphone, hermes_gateway, $lineColor="#238636", $textColor="#238636")
    UpdateRelStyle(macbook, zed, $lineColor="#f85149", $textColor="#f85149")
```
