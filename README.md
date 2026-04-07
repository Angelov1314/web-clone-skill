# web-clone — Claude Code Skill

A Claude Code skill for pixel-perfect website cloning. Uses a 5-phase pipeline combining browser automation, real CSS extraction via `getComputedStyle`, and parallel builder agents to reproduce any website with high fidelity.

## Install

Drop `SKILL.md` into your Claude Code skills directory (`.claude/skills/` or as configured).

## Usage

```
/web-clone <url>
/web-clone <url> <output-dir>
/web-clone survey <url>     # Phase 1 only: recon + design tokens
/web-clone audit <url>      # Phase 1+2: survey + interactions
/web-clone qa               # Phase 5: visual QA on current clone
```

## Pipeline Phases

| Phase | Name | What it does |
|-------|------|--------------|
| 1 | **Survey** | Navigate, screenshot, extract fonts/colors/layout from computed styles |
| 2 | **Interaction Sweep** | Record hover, scroll, click, and animation behaviors |
| 3 | **Spec Writing** | Generate per-component spec files for builder agents |
| 4 | **Parallel Build** | Dispatch multiple worktree agents to build components simultaneously |
| 5 | **Visual QA** | Side-by-side pixel diff against original screenshots |

## Core Philosophy

> Extraction over guessing. Specs over vibes. Parallel over serial.

- Every CSS value comes from `getComputedStyle()` — never estimated
- Each builder agent receives a complete spec before writing any code
- Foundation (fonts, colors, globals) is built sequentially; components in parallel

## Prerequisites

- **Browser automation MCP**: Claude in Chrome (preferred), Playwright, or Puppeteer
- **Claude Preview MCP**: for build verification

## Default Output Stack

- Next.js 15+ with App Router + TypeScript
- Tailwind CSS v4 with oklch design tokens
- shadcn/ui (Radix primitives)
- Lucide React

A different stack can be specified at invocation time.
