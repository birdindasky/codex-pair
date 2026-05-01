<div align="center">

<img src="assets/banner.svg" alt="codex-pair — Two AIs review each other so you don't have to" width="100%" />

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Built with Claude Code](https://img.shields.io/badge/built%20with-Claude%20Code-FF6B35)](https://claude.com/claude-code)
[![Status: experimental](https://img.shields.io/badge/status-experimental-yellow)](#)

**English** | [中文](./README_zh.md)

</div>

> You asked Claude to fix the bug. Claude said done. You ship it.
> An hour later you find out it's still broken — and you couldn't have known, because you can't read the code.
>
> **This skill makes that scenario impossible.**

## ✨ How it feels

```
┌─ Claude Code ─────────────────────────────────────────────────────┐
│                                                                   │
│  You      Make me a CLI that converts PDF to Markdown.            │
│           Use codex-pair on this.                                 │
│                                                                   │
│  Claude   Two paths — A) PyMuPDF + custom writer                  │
│                       B) Marker (heavier, better tables)          │
│           Which?                                                  │
│                                                                   │
│  You      A.                                                      │
│                                                                   │
│           ⋯  20 minutes of two AIs working  ⋯                    │
│                                                                   │
│  Claude   Done.                                                   │
│           ✓ Codex caught 2 bugs during review (looped once)      │
│           ✓ Tests pass · I ran it on a real PDF · works          │
│           ✓ Committed as a3f9b2e                                 │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

You touched the keyboard **twice** — once to start, once to pick A.

## 🧭 The pipeline

```mermaid
flowchart TD
    A([🟦 1·Architecture]) --> B([🟦 2·Plan + Interface contract])
    B --> C([🟧 3·Codex reviews])
    C --> D{4·Consensus?<br/>max 3 rounds}
    D -->|Agreed| E([🟧 5·Codex implements])
    D -->|Deadlock| Y[👤 You break the tie]
    Y --> E
    E --> F([🟧 6·Code-level tests])
    F --> G([🟦 7·Behavior verification])
    G -->|Code bug| E
    G -->|Plan gap| B
    G -->|Works| H([🟦 8·Commit])

    style A fill:#dbeafe,stroke:#1e40af,color:#000
    style B fill:#dbeafe,stroke:#1e40af,color:#000
    style G fill:#dbeafe,stroke:#1e40af,color:#000
    style H fill:#dcfce7,stroke:#15803d,color:#000
    style C fill:#fed7aa,stroke:#c2410c,color:#000
    style E fill:#fed7aa,stroke:#c2410c,color:#000
    style F fill:#fed7aa,stroke:#c2410c,color:#000
    style Y fill:#fef3c7,stroke:#b45309,color:#000
```

🟦 Claude · 🟧 Codex · 👤 You (only on real deadlocks)

## 🚀 Install

```bash
git clone https://github.com/birdindasky/codex-pair ~/codex-pair-src
cp -r ~/codex-pair-src/skills/codex-pair ~/.claude/skills/
```

Open Claude Code, type `/`, look for `codex-pair`.

**Needs:** [Claude Code](https://claude.com/claude-code) + [`openai/codex-plugin-cc`](https://github.com/openai/codex-plugin-cc) plugin. Run `/codex:setup` once if Codex isn't set up yet.

## 🎬 Trigger phrases

In any Claude Code conversation, say one of:

- `codex pair this`
- `two-AI mode`
- `use codex on this`

Mid-pipeline overrides: `skip codex` (cancel) · `skip review` (bypass review) · `run them in parallel` (switch to worktree mode).

## ⚖️ Tradeoffs

Costs roughly **2× tokens** of solo Claude · **slower** than solo (consensus loop adds time) · **won't save you from wrong requirements** at step 1. For typo fixes or one-line tweaks, just ask Claude directly. This pipeline is for **projects**.

## 📂 Files

- [`skills/codex-pair/SKILL.md`](skills/codex-pair/SKILL.md) — the actual skill
- [`PROTOCOLS.md`](PROTOCOLS.md) — worklog · pending-commits · venv specs
- [`examples/walk-through.md`](examples/walk-through.md) — full end-to-end example: building a PDF→Markdown CLI
- [`LICENSE`](LICENSE) — MIT

Inspired by [Matt Pocock's skills repo](https://github.com/mattpocock/skills). Pipeline design is this project's own.

---

<div align="center">

**Built by [@birdindasky](https://github.com/birdindasky) · For everyone who codes by vibes.**

⭐ Star if this saves you from one silent AI bug.

</div>
