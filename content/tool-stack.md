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

- Linux containers on macOS
- Occasional use

**[uv](https://docs.astral.sh/uv/)**

- Python project and environment management
- Replacing `pip`, `conda` or `pyenv`

## Editing

**[Neovim](https://neovim.io/)**

- Lua configuration in my dotfiles repository
- Python: basedpyright and Ruff
- Markdown: marksman and render-markdown
- LaTeX: VimTeX + texlab; Tectonic builds on save
- blink.cmp completion
- SSH + tmux remote editing, with OSC52 clipboard
- WakaTime activity tracking

**[Obsidian](https://obsidian.md/)**

- Markdown-based knowledge management
- Tech notes, research notes, and personal documentation
- Publish notes to [my personal website](https://allenyolk.github.io/) through **[Quartz](https://quartz.jzhao.xyz/)**

**[Oleafly](https://oleafly.com/)**

- Local LaTeX editor and PDF previewer, currently evaluating alongside Neovim + VimTeX
- Keeping standalone Tectonic alongside it for now

**[tectonic](https://tectonic-typesetting.github.io/)**

- Self-contained LaTeX engine
- Fetches LaTeX packages on demand

## Terminal

**[Ghostty](https://ghostty.org/)**

- GPU-accelerated terminal emulator
- Modern features & smooth rendering

**[herdr](https://github.com/herdrdev/herdr)**

- Terminal sessions for CLI agents
- I use it heavily and highly recommend it
- Persistent sessions for remote work

**[Zsh](https://www.zsh.org/) + [Starship](https://starship.rs/)**

- Native Zsh configuration with a Starship prompt
- Standalone autosuggestions and syntax highlighting
- Sync dotfiles through GitHub

## AI

**[OpenBot](https://openbot.run/)**

- Persistent local agents with dedicated roles and workspaces
- Task routing and handoffs between agents, delegating coding work to T3 Code via MCP

**[T3 Code](https://t3.codes/)**

- Primary desktop interface for daily coding tasks and reviewing diffs
- Connects Codex, Antigravity, and Pi in one workspace, with MCP access from OpenBot

**[Codex](https://openai.com/codex/)**

- Coding agent used through OpenBot, T3 Code, and the CLI
- Refactoring, code review, and documentation
- [Remote](https://learn.chatgpt.com/docs/remote) through the ChatGPT mobile app to my connected Mac
- No Tailscale needed for Codex Remote; the Mac must stay awake and online
- MiniMax is also configured, but rarely used here

**[Antigravity](https://antigravity.google/)**

- Gemini-backed agent runtime enabled in OpenBot and T3 Code
- Local `agy` CLI installed alongside it

**[Pi](https://pi.dev/)**

- My CLI entry point for non-GPT models, also connected to T3 Code
- Transparent internals and customizable plugins
- I build plugins and contribute to the community
- gotgenes' subagent plugin for delegated tasks
- [@allenyolk/pi-minimal-display](https://github.com/AllenYolk/pi-extensions/tree/main/packages/pi-minimal-display): compact tool output with expandable details
- [@allenyolk/pi-delete](https://github.com/AllenYolk/pi-extensions/tree/main/packages/pi-delete): session and subtree deletion
- [@allenyolk/pi-compaction](https://github.com/AllenYolk/pi-extensions/tree/main/packages/pi-compaction): custom compaction model and thinking level, with whole-summary persistence

**Providers**

- ChatGPT (OpenAI)
- Google / Gemini (via Antigravity)
- MiniMax M-plan

## Network

**[Tailscale](https://tailscale.com/)**

- Private networking across my devices
- Tailscale + [macOS Screen Sharing](https://support.apple.com/en-sg/guide/mac-help/mh14066/mac): Macbook to Mac Mini
- Tailscale + SSH: direct access to Pi and other terminal tools on my Mac Mini
- Ghostty on my Macbook; [Termius](https://termius.com/) on my iPhone
- SSH + Neovim keeps repositories on the Mac Mini

## Summary

```mermaid
flowchart TB
    phone["iPhone"] --> remote["ChatGPT / Codex Remote"]
    remote -->|"No Tailscale required"| codex["Codex on Mac Mini"]
    macbook["Macbook"] -->|"Tailscale + SSH"| terminal["herdr on Mac Mini"]
    phone -->|"Termius + Tailscale + SSH"| terminal
    terminal --> pi["Pi"]
    terminal --> nvim["Neovim"]
    terminal --> cli["Codex CLI"]
    openbot["OpenBot agents"] --> cli
    openbot --> agy["Antigravity"]
    openbot -->|"MCP"| t3["T3 Code"]
    t3 --> cli
    t3 --> agy
    t3 --> pi
    codex --> openai["ChatGPT / OpenAI"]
    cli --> openai
    agy --> gemini["Google / Gemini"]
    codex -.->|"Rarely used"| minimax["MiniMax M-plan"]
    pi --> minimax
```
