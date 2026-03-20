# How the Excalidraw Diagram Skill Works

This document demystifies the "magic" behind the [excalidraw-diagram-skill](https://github.com/coleam00/excalidraw-diagram-skill) for Claude Code. There are three layers: the **methodology** (how Claude thinks about diagrams), the **rendering pipeline** (how JSON becomes pixels), and the **research trigger** (when and why Claude goes looking things up before drawing).

---

## 1. The Methodology: Diagrams That Argue

The core skill file (`SKILL.md`, ~25 KB) is not code — it's a detailed instruction manual that shapes how Claude approaches diagram creation. Its central thesis:

> **Diagrams should ARGUE, not DISPLAY.**

This means every diagram must pass two tests:

- **The Isomorphism Test**: If you removed all text, would the visual structure alone communicate the concept? (Fan-outs show one-to-many, convergence shows many-to-one, timelines show sequence.)
- **The Education Test**: Could someone learn something concrete from this diagram that they couldn't learn from a bullet list?

### Visual Pattern Library

The skill defines 8 visual patterns that Claude maps concepts onto:

| Pattern | When to Use | Example |
|---------|------------|---------|
| **Fan-Out** | One thing produces many | API Gateway → multiple services |
| **Convergence** | Many things merge into one | Parallel CI jobs → merge gate |
| **Assembly Line** | Step-by-step transformation | Input → Process → Output |
| **Tree** | Hierarchies and branching | Decision trees, org charts |
| **Spiral/Cycle** | Recurring processes | React render loop, retry logic |
| **Side-by-Side** | Comparisons | Event Sourcing vs CRUD |
| **Cloud** | Abstract/fuzzy states | Overlapping ellipses for context |
| **Gap/Break** | Phase transitions | Visual whitespace between stages |

### Multi-Zoom Architecture

Every comprehensive diagram has three simultaneous levels of detail:

1. **Level 1 — Summary flow**: The big picture (top or bottom of canvas)
2. **Level 2 — Section boundaries**: Labeled regions grouping related components
3. **Level 3 — Detail**: Evidence artifacts, code snippets, concrete data inside sections

### Shape Semantics

Shapes carry meaning — they're not decorative:

- **No container** (free-floating text): Labels, descriptions, details — the default (>70% of text)
- **Rectangle**: Process, action, step
- **Ellipse**: Start/trigger, input/output, external system
- **Diamond**: Decision, condition, branch point
- **Lines + text**: Hierarchy/tree structure
- **Overlapping ellipses**: Abstract state or context

### Color Palette

All colors come from a single file (`color-palette.md`) — Claude never invents colors. Each color has semantic meaning:

| Purpose | Fill | Stroke |
|---------|------|--------|
| Primary/Neutral | `#3b82f6` | `#1e3a5f` |
| Start/Trigger | `#fed7aa` | `#c2410c` |
| End/Success | `#a7f3d0` | `#047857` |
| Warning/Error | `#fee2e2` / `#fecaca` | `#dc2626` / `#b91c1c` |
| Decision | `#fef3c7` | `#b45309` |
| AI/LLM | `#ddd6fe` | `#6d28d9` |

Evidence artifacts (code blocks, JSON payloads) use a dark background (`#1e293b`) with green (`#22c55e`) or syntax-colored text — mimicking a terminal or IDE.

---

## 2. The Rendering Pipeline: JSON → PNG

This is where Playwright and Chromium enter the picture. The skill doesn't draw pixels directly — it generates Excalidraw JSON, then uses a headless browser to render it.

### The Architecture

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌─────────┐
│ Claude       │────▶│ .excalidraw JSON │────▶│ render_excalidraw│────▶│  .png   │
│ generates    │     │ file             │     │ .py              │     │  file   │
│ JSON         │     └──────────────────┘     └────────┬─────────┘     └─────────┘
└─────────────┘                                        │
                                                       │ launches
                                                       ▼
                                              ┌──────────────────┐
                                              │ Headless Chromium │
                                              │ via Playwright    │
                                              │                   │
                                              │ ┌───────────────┐│
                                              │ │render_template││
                                              │ │.html          ││
                                              │ │               ││
                                              │ │imports        ││
                                              │ │@excalidraw/   ││
                                              │ │excalidraw from││
                                              │ │esm.sh CDN    ││
                                              │ │               ││
                                              │ │exportToSvg() ││
                                              │ └───────────────┘│
                                              └──────────────────┘
```

### Step-by-Step Rendering Flow

**Step 1 — Validate JSON** (`render_excalidraw.py`)

The Python script reads the `.excalidraw` file and checks:
- Is it valid JSON?
- Does it have `"type": "excalidraw"`?
- Does it have an `elements` array?

**Step 2 — Compute Bounding Box**

The script calculates the bounding box of all elements (min/max x, y, width, height) and adds 80px padding. This determines the browser viewport size so nothing gets clipped.

**Step 3 — Launch Headless Chromium**

Playwright launches a headless (no visible window) Chromium browser instance:
- Viewport sized to fit the diagram
- Device scale factor of 2x for sharp output

**Step 4 — Load the HTML Template**

The browser navigates to `render_template.html` via a `file://` URL. This HTML page:
- Imports `exportToSvg` from the official Excalidraw library via the [esm.sh](https://esm.sh) CDN
- Sets `window.__moduleReady = true` when the import completes

**Step 5 — Inject Diagram Data**

The Python script calls `window.renderDiagram(jsonString)` inside the browser. This JavaScript function:
1. Parses the JSON
2. Forces white background (`#ffffff`)
3. Calls `exportToSvg()` — the same function Excalidraw's web app uses internally
4. Appends the generated SVG to the DOM
5. Sets `window.__renderComplete = true`

**Step 6 — Screenshot**

Playwright selects the SVG element by CSS selector and takes a screenshot of just that element, saving it as a PNG.

### Why Playwright + Chromium?

Excalidraw's rendering engine is a JavaScript library designed to run in a browser. There is no native Python or CLI renderer. The only way to convert Excalidraw JSON to an image is to:

1. Run the actual Excalidraw JavaScript library
2. Which requires a browser DOM (for SVG rendering)
3. Which requires a browser engine

**Playwright** is the bridge — it provides a Python API to control a real browser programmatically. **Chromium** is the browser engine that actually executes the Excalidraw JavaScript and renders the SVG.

This is the same technique used by tools like Puppeteer, but Playwright is preferred here because:
- It has first-class Python support (no Node.js dependency for the script itself)
- It handles browser installation (`playwright install chromium`)
- It provides element-level screenshots (screenshot just the SVG, not the whole page)

### Setup Requirements

```bash
cd ~/.claude/skills/excalidraw-diagram-skill/references
uv sync                              # Install Python dependencies (playwright)
uv run playwright install chromium    # Download Chromium binary (~150 MB)
```

After this, rendering is fully local — no API calls, no cloud services.

---

## 3. The Research Trigger: When Claude Looks Things Up

Not every diagram triggers research. The skill defines a **depth assessment** that determines Claude's approach:

### Simple/Conceptual Diagrams (No Research Needed)

If the topic is:
- Well-known and generic (e.g., "client-server architecture")
- Conceptual rather than spec-specific (e.g., "how microservices communicate")
- Something Claude already knows deeply

Then Claude skips research and goes straight to design.

### Comprehensive/Technical Diagrams (Research Required)

If the topic involves:
- **Specific protocols or specs** (e.g., AG-UI protocol, OAuth 2.0 flows)
- **Real event names, API formats, or data structures** (e.g., actual SSE event types)
- **Named tools or frameworks** with specific APIs (e.g., Temporal workflows, Kubernetes scheduling)
- **Anything where getting the details wrong would make the diagram misleading**

Then Claude **must research first**. The skill explicitly states:

> For technical diagrams, the agent should research actual specs, event names, JSON formats, and real-world data before designing. Evidence artifacts (code snippets, data examples, real input content) prove accuracy.

### How Research Happens

Claude uses its available tools to research:
1. **Web search** for official documentation, specs, and RFCs
2. **Web fetch** to read specific documentation pages
3. **Codebase exploration** if the topic relates to the current project

For example, in the AG-UI protocol diagram from this repo, Claude:
1. Searched for the AG-UI protocol specification
2. Found the actual event type names (`RUN_STARTED`, `TEXT_MESSAGE_CONTENT`, `TOOL_CALL_START`, etc.)
3. Found the real SSE wire format (`data: {"type":"RUN_STARTED",...}`)
4. Found the TypeScript client API (`HttpAgent`, `.subscribe()`)
5. Only then designed the diagram with accurate event names and payload shapes

The transformer attention diagram, by contrast, needed no research — Claude already knows the mechanism deeply.

### The Evidence Artifact Requirement

Research feeds directly into **evidence artifacts** — concrete, real-world data embedded in the diagram:

- Code snippets with actual syntax
- JSON payloads with real field names
- Data tables with plausible values
- Terminal output or wire formats

These artifacts are what make skill-generated diagrams more useful than generic flowcharts. They transform a diagram from "here's roughly how it works" into "here's exactly how it works, with proof."

---

## 4. The Build-Render-Validate Loop

The final piece of the puzzle is the mandatory validation cycle:

```
Design → Generate JSON → Render PNG → View PNG → Audit → Fix → Re-render → ...
```

Claude doesn't just generate JSON and ship it. The skill requires:

1. **Render** the diagram to PNG using the Playwright pipeline
2. **View** the PNG (Claude can read images)
3. **Audit** against the original design vision:
   - Does the visual structure match the concept?
   - Are patterns correct (fan-out where intended, convergence where intended)?
   - Is text readable, not clipped or overlapping?
   - Do arrows route cleanly?
   - Is spacing consistent?
4. **Fix** any issues in the JSON
5. **Re-render** and re-validate

This loop typically runs 2-4 times. It catches problems that are invisible in raw JSON but obvious visually — text overflow, overlapping elements, arrows pointing to wrong targets, unbalanced layouts.

### Large Diagram Strategy

For complex diagrams that exceed Claude's output token limit (~32K tokens):

1. Build JSON **section by section** (not all at once)
2. Use descriptive string IDs (`"trigger_rect"`, `"arrow_fan_left"`) instead of random numbers
3. Namespace random seeds by section (section 1: `100xxx`, section 2: `200xxx`)
4. Add cross-section arrows after all sections are built
5. Render and validate the complete diagram

---

## Summary

| Layer | What It Does | Key Files |
|-------|-------------|-----------|
| **Methodology** | Teaches Claude *how to think* about diagrams | `SKILL.md`, `color-palette.md`, `element-templates.md` |
| **Rendering** | Converts JSON to PNG via headless browser | `render_excalidraw.py`, `render_template.html`, Playwright + Chromium |
| **Research** | Ensures technical accuracy before drawing | Triggered by depth assessment in `SKILL.md` |
| **Validation** | Catches visual bugs through render-view-fix loop | Render pipeline + Claude's image reading |

The "magic" is really three things working together: a detailed methodology that produces good designs, a browser-based pipeline that makes them visible, and a validation loop that catches mistakes before delivery.
