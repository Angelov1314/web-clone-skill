---
name: web-clone
description: "Reverse-engineer and pixel-perfect clone any website using browser automation (Claude in Chrome MCP) and preview verification. Extracts real CSS via getComputedStyle, downloads all assets, writes component specs, and dispatches parallel builder agents in worktrees. Triggers on: clone website, replicate site, copy page, pixel-perfect clone, rebuild this page."
argument-hint: "<target-url> [output-dir]"
user-invocable: true
triggers:
  - clone
  - clone website
  - web clone
  - replicate site
  - copy page
  - pixel-perfect
  - rebuild page
  - reverse engineer site
---

# Web Clone

From URL to pixel-perfect reproduction. Multi-phase pipeline with parallel builder agents.

## Commands

- **`/web-clone <url>`** — Full clone pipeline (all 5 phases)
- **`/web-clone <url> <output-dir>`** — Clone into specific directory
- **`/web-clone survey <url>`** — Phase 1 only: reconnaissance + design token extraction
- **`/web-clone audit <url>`** — Phase 1+2: survey + interaction sweep
- **`/web-clone qa`** — Phase 5 only: run visual QA on current clone project

## Core Philosophy

> **Extraction over guessing. Specs over vibes. Parallel over serial.**

- Every CSS value comes from `getComputedStyle()` — never estimate
- Every builder agent receives a complete spec file before writing code
- Foundation (fonts, colors, globals) is sequential; everything else is parallel
- Extract appearance AND behavior (hover, scroll, click, animation)
- Small focused tasks yield perfect results; split specs >150 lines

## Prerequisites

### Required MCP Tools

This skill requires **browser automation MCP**. Detection priority:
1. **Claude in Chrome** (`mcp__Claude_in_Chrome__*`) — preferred
2. **Playwright MCP** — fallback
3. **Puppeteer MCP** — fallback

If none detected, ask the user to configure one.

### Required Preview Tools

**Claude Preview** (`mcp__Claude_Preview__*`) for verifying the clone build.

### Tech Stack (Default)

- **Next.js 15+** with App Router + TypeScript
- **Tailwind CSS v4** with oklch design tokens
- **shadcn/ui** (Radix primitives)
- **Lucide React** (replaced by extracted SVGs during clone)

If user specifies a different stack, adapt accordingly.

---

## Phase 1: SURVEY (Reconnaissance)

### Step 1.1 — Navigate & Screenshot

```
1. Open target URL via browser automation
2. Capture full-page screenshots at:
   - Desktop: 1440px width
   - Tablet: 768px width
   - Mobile: 390px width
3. Save to docs/design-references/
```

### Step 1.2 — Extract Global Design Tokens

**Fonts:** Inspect `<link>` tags + computed `font-family` on headings, body, code, labels.

Run via browser JS:
```javascript
JSON.stringify({
  fonts: [...new Set([...document.querySelectorAll('*')].map(el =>
    getComputedStyle(el).fontFamily).filter(f => f))],
  headingFont: getComputedStyle(document.querySelector('h1,h2,h3'))?.fontFamily,
  bodyFont: getComputedStyle(document.body).fontFamily,
  fontLinks: [...document.querySelectorAll('link[href*="font"]')].map(l => l.href)
});
```

**Colors:** Extract full palette from computed styles:
```javascript
JSON.stringify([...new Set([...document.querySelectorAll('*')].flatMap(el => {
  const cs = getComputedStyle(el);
  return [cs.color, cs.backgroundColor, cs.borderColor].filter(c => c && c !== 'rgba(0, 0, 0, 0)');
}))]);
```

**Favicons & Meta:** Download favicons, OG images, webmanifest to `public/seo/`.

**Global patterns:** Check for smooth scroll libraries (Lenis, Locomotive), scroll-snap, custom scrollbars, global keyframe animations, backdrop filters.

### Step 1.3 — Page Topology Mapping

Map every section top-to-bottom. For each document:
- Visual order and nesting
- Fixed/sticky overlays vs flow content
- z-index layers
- Dependencies between sections
- **Interaction model**: static / click-driven / scroll-driven / time-driven

Save as `docs/research/PAGE_TOPOLOGY.md`.

---

## Phase 2: INTERACTION SWEEP

> This phase discovers every behavior. Many are invisible in screenshots.

### Step 2.1 — Scroll Sweep

Scroll slowly top to bottom. At each section pause and observe:
- Header appearance changes? Record scroll position
- Elements animate into view? Record which + animation type
- Sidebar/tab auto-switches on scroll? Record mechanism
- Scroll-snap points? Record containers
- Smooth scroll library active? Check for non-native behavior

### Step 2.2 — Click Sweep

Click every interactive element: buttons, tabs, pills, links, cards.
- For tabs/pills: click EACH ONE, record content per state
- For dropdowns: open each, capture options
- For modals: trigger and screenshot content

### Step 2.3 — Hover Sweep

Hover every element with potential hover states. Record changes:
- Color, scale, shadow, underline, opacity, transform
- Transition timing and easing

### Step 2.4 — Responsive Sweep

Test at 1440px, 768px, 390px via browser resize:
- Which sections change layout?
- Approximate breakpoint trigger points
- Navigation collapse behavior
- Hidden/shown elements per viewport

Save all findings to `docs/research/BEHAVIORS.md`.

---

## Phase 3: FOUNDATION BUILD

**Sequential — do NOT delegate to agents.**

1. **Scaffold project** (if not exists):
   ```bash
   npx create-next-app@latest <project-name> --typescript --tailwind --app --src-dir
   npx shadcn@latest init
   ```

2. **Configure fonts** in `src/app/layout.tsx` using `next/font/google` or `next/font/local`

3. **Write color tokens** to `src/app/globals.css`:
   - Map to shadcn token names where possible
   - Add custom properties for unmapped colors
   - Include `:root` and `.dark` blocks

4. **Extract SVG icons** — deduplicate, save as React components in `src/components/icons.tsx`

5. **Download all assets** — write `scripts/download-assets.mjs`:
   ```javascript
   // Run in browser to enumerate assets
   JSON.stringify({
     images: [...document.querySelectorAll('img')].map(img => ({
       src: img.src || img.currentSrc,
       alt: img.alt,
       width: img.naturalWidth,
       height: img.naturalHeight
     })),
     videos: [...document.querySelectorAll('video')].map(v => ({
       src: v.src || v.querySelector('source')?.src,
       poster: v.poster, autoplay: v.autoplay, loop: v.loop
     })),
     backgroundImages: [...document.querySelectorAll('*')].filter(el => {
       const bg = getComputedStyle(el).backgroundImage;
       return bg && bg !== 'none';
     }).map(el => ({
       url: getComputedStyle(el).backgroundImage,
       element: el.tagName + '.' + (el.className?.split?.(' ')[0] || '')
     }))
   });
   ```
   Download to `public/images/`, `public/videos/`, `public/fonts/`

6. **Verify:** `npm run build` passes

---

## Phase 4: PARALLEL BUILD

### Step 4.1 — Write Component Specs

For each section in PAGE_TOPOLOGY.md, create `docs/research/components/<name>.spec.md`:

```markdown
# <ComponentName> Specification

## Overview
- **Target file:** `src/components/<ComponentName>.tsx`
- **Screenshot:** `docs/design-references/<screenshot>.png`
- **Interaction model:** <static | click-driven | scroll-driven | time-driven>

## DOM Structure
<Element hierarchy>

## Computed Styles (exact getComputedStyle values)

### Container
- display: flex
- padding: 64px 80px
- maxWidth: 1280px
- backgroundColor: rgb(255, 255, 255)

### <Child elements with full CSS>

## States & Behaviors

### <Behavior name>
- **Trigger:** <scroll position / click / hover / IntersectionObserver>
- **State A:** <CSS values before>
- **State B:** <CSS values after>
- **Transition:** <transition property>

## Per-State Content (for tabs/carousels)
### State: "<Tab Name>"
- Content: [{ title, description, image, link }, ...]

## Assets
- Images: `public/images/<file>`
- Icons: <IconName> from icons.tsx

## Text Content (verbatim from site)
<Exact text, not paraphrased>

## Responsive Behavior
- **Desktop (1440px):** <layout>
- **Tablet (768px):** <changes>
- **Mobile (390px):** <changes>
```

### Step 4.2 — Extract CSS for Specs

Use this extraction script via browser automation for each section:

```javascript
(function(selector) {
  const el = document.querySelector(selector);
  if (!el) return JSON.stringify({ error: 'Not found: ' + selector });
  const props = [
    'fontSize','fontWeight','fontFamily','lineHeight','letterSpacing','color',
    'textTransform','textDecoration','backgroundColor','background',
    'padding','paddingTop','paddingRight','paddingBottom','paddingLeft',
    'margin','marginTop','marginRight','marginBottom','marginLeft',
    'width','height','maxWidth','minWidth','maxHeight','minHeight',
    'display','flexDirection','justifyContent','alignItems','gap',
    'gridTemplateColumns','gridTemplateRows',
    'borderRadius','border','boxShadow','overflow',
    'position','top','right','bottom','left','zIndex',
    'opacity','transform','transition','cursor',
    'objectFit','objectPosition','mixBlendMode','filter','backdropFilter',
    'whiteSpace','textOverflow','WebkitLineClamp'
  ];
  function extractStyles(element) {
    const cs = getComputedStyle(element);
    const styles = {};
    props.forEach(p => { const v = cs[p]; if (v && v !== 'none' && v !== 'normal' && v !== 'auto' && v !== '0px') styles[p] = v; });
    return styles;
  }
  function walk(element, depth) {
    if (depth > 4) return null;
    return {
      tag: element.tagName.toLowerCase(),
      classes: element.className?.toString().split(' ').slice(0, 5).join(' '),
      text: element.childNodes.length === 1 && element.childNodes[0].nodeType === 3
        ? element.textContent.trim().slice(0, 200) : null,
      styles: extractStyles(element),
      images: element.tagName === 'IMG' ? { src: element.src, alt: element.alt } : null,
      children: [...element.children].slice(0, 20).map(c => walk(c, depth + 1)).filter(Boolean)
    };
  }
  return JSON.stringify(walk(el, 0), null, 2);
})('SELECTOR_HERE');
```

### Step 4.3 — Dispatch Builder Agents

**Complexity rule:**
- Simple section (1-2 sub-components) → 1 builder agent
- Complex section (3+ sub-components OR spec >150 lines) → split into multiple agents

**Each builder agent receives:**
- Full spec file contents inline in the prompt
- Path to section screenshot
- Shared component imports list
- Target file path
- Instruction: verify with `npx tsc --noEmit` before finishing

**Dispatch pattern using Agent tool:**
```
Agent(
  subagent_type: "general-purpose",
  isolation: "worktree",
  prompt: "Build <ComponentName> following this exact specification: <spec contents>..."
)
```

Dispatch builders for completed specs immediately. Don't wait — extract next section while builders work.

### Step 4.4 — Merge

As builders complete:
1. Merge worktree branches into main
2. Resolve conflicts using full context
3. After each merge: `npm run build`
4. Fix type errors immediately

---

## Phase 5: FIDELITY CHECK (Visual QA)

### Step 5.1 — Side-by-Side Comparison

1. Start dev server: `npm run dev`
2. Open clone via Preview tools
3. Compare section-by-section against original at 1440px
4. Compare again at 390px

### Step 5.2 — Discrepancy Resolution

For each difference found:
- Check component spec — was the value extracted correctly?
- If spec wrong → re-extract via browser, update spec, fix component
- If spec right but builder wrong → fix component to match spec

### Step 5.3 — Interaction Verification

- Scroll through entire page — verify animations, header transitions
- Click every tab/button — verify state changes
- Hover interactive elements — verify effects
- Test responsive at 1440 / 768 / 390

### Step 5.4 — Completion Report

```markdown
## Clone Report
- **Source:** <url>
- **Sections built:** N
- **Components created:** N
- **Spec files written:** N
- **Assets downloaded:** N (images, videos, SVGs, fonts)
- **Build status:** npm run build ✓/✗
- **Visual QA:** <pass / N discrepancies remaining>
- **Known gaps:** <any limitations>
```

---

## Pre-Dispatch Checklist

Before dispatching ANY builder agent, verify:

- [ ] Spec file written to `docs/research/components/<name>.spec.md`
- [ ] Every CSS value from `getComputedStyle()`, not estimated
- [ ] Interaction model identified (static / click / scroll / time)
- [ ] All states captured (tabs clicked, scroll positions tested)
- [ ] Hover before/after values + transition timing recorded
- [ ] All images identified (including overlays, layered backgrounds)
- [ ] Responsive behavior documented for desktop + mobile
- [ ] Text content verbatim from site
- [ ] Spec under ~150 lines; if over, split the section

## Anti-Patterns (What NOT To Do)

- **Don't build click tabs when original is scroll-driven** — determine interaction model FIRST
- **Don't extract only default state** — click every tab, scroll to every trigger
- **Don't miss overlay/layered images** — check full DOM tree per container
- **Don't build HTML for video/Lottie/canvas content** — detect media type first
- **Don't approximate CSS** — "looks like text-lg" is wrong if computed value differs
- **Don't skip asset download** — without real images, clone looks fake
- **Don't give builders >150 lines of spec** — split into focused sub-tasks
- **Don't dispatch without spec files** — specs force exhaustive extraction
- **Don't skip responsive extraction** — test at 1440, 768, 390 during extraction
- **Don't forget smooth scroll libraries** — Lenis/Locomotive feel different from native
