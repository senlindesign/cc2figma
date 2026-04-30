# Claude Code to Figma

[![Made for Claude Code](https://img.shields.io/badge/Made%20for-Claude%20Code-blueviolet?style=flat-square&logo=anthropic)](https://docs.anthropic.com/en/docs/claude-code)
[![Figma MCP](https://img.shields.io/badge/Figma-MCP%20Server-ff7262?style=flat-square&logo=figma)](https://www.npmjs.com/package/@anthropic-ai/figma-mcp)
[![License: MIT](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

**A set of Claude Code Skills that make AI build Figma designs following your Design System — components linked to masters, visual values bound to tokens, zero hardcoded values.**

[Chinese / 中文版](README.zh-CN.md)

---

## Before & After

### Without Skills — Hardcoded values, no bindings

<!-- Replace with actual screenshot -->
![Without Skills](assets/without-skills.png)

> Components built from scratch. Colors, fonts, and spacing are raw values with no Design System connection.

### With Skills — Component links, token bindings

<!-- Replace with actual screenshot -->
![With Skills](assets/with-skills.png)

> Components are Instances of Master Components. All colors, fonts, spacing, and radii are bound to Design System Variables and Styles.

---

## Preflight System

Before every design session, 7 automated checks ensure everything is connected:

<!-- Replace with actual screenshot -->
![Preflight Status](assets/preflight-status.png)

---

## What It Does

- **One-click preflight** — Verifies MCP connection, loads complete Styles / Variables / Components inventory
- **Components first** — Searches the Design System and creates Instances of Master Components
- **Token-bound design** — Colors, fonts, spacing, and radii always bind to Variables / Styles — no hardcoded values
- **Auto QA** — Verifies style/variable bindings on every node after each Figma write
- **Reference interpreter** — Give it a screenshot or URL, get a structured Design Brief

## Good For / Not For

| Good For | Not For |
|----------|---------|
| Building pages from an existing Design System | Creating a Design System from scratch |
| Describing UI in natural language, Claude builds it in Figma | Pixel-perfect illustration or icon drawing |
| Working in a DS file or a file with a linked DS library | Free-form design without a Design System |
| Ensuring 100% token compliance in design output | FigJam / whiteboard workflows |

---

## Installation

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (CLI / Desktop / VS Code)
- [Figma MCP Server](https://www.npmjs.com/package/@anthropic-ai/figma-mcp) installed and authenticated
- A Figma file with a Design System (locally defined or linked via Library)

### Install

```bash
# 1. Clone the repo
git clone https://github.com/senlindesign/cc2figma.git

# 2. Copy skills to your project
cp -r cc2figma/.claude/skills/* your-project/.claude/skills/

# 3. Copy config
cp cc2figma/.claude/settings.json your-project/.claude/settings.json

# 4. Copy CLAUDE.md template
cp cc2figma/CLAUDE.md.template your-project/CLAUDE.md
```

### Configure CLAUDE.md

Open `CLAUDE.md` and fill in your Figma file details:

```markdown
# Figma Design Project

- **Figma file:** <https://www.figma.com/design/YOUR_FILE_KEY/...>
- **Fonts:** Inter (primary) · Roboto Mono (code)
- **Session goal:** [what are we designing today?]

## Rules

1. Every visual value must bind to a Style or Variable.
2. Always search connected libraries before building any component from scratch.
3. Never start designing before the Design Brief is confirmed.
```

> The Fonts field can be left blank — Preflight will auto-populate it from Design System Variables.

---

## Workflow

```
1. Open Claude Code in your project directory
2. Say "Let's start"                          → triggers Preflight
3. Wait for all 7 checks to pass              → Styles / Variables / Components loaded
4. Describe the design                        → "Build a login page with email and password"
5. Claude searches DS components → creates Instances → binds Tokens
6. Screenshot verification after each step     → confirm and continue
7. Result: Figma design with 100% token binding and proper component links
```

---

## Skills Reference

| Skill | Trigger | Responsibility |
|-------|---------|---------------|
| `figma-preflight` | "let's start", first Figma URL shared | 7 checks + Token Map + Component Registry |
| `component-rules` | "create a card", "build a button", any UI construction | Library-first lookup, Auto Layout, semantic naming |
| `figma-style-binding` | Any color, font, or spacing operation | Enforce Variable / Style binding on all visual values |
| `figma-qa-verifier` | Auto-triggers after every `use_figma` call | Check binding status, report raw values |
| `reference-interpreter` | Share screenshot, URL, or design description | Output structured Design Brief |

### How They Work Together

```
Session start
    |
    v
+-----------------+
|  figma-preflight | <-- Load Token Map + Component Registry
+--------+--------+
         |
         v
+-----------------+     +---------------------+
| component-rules  | --> |  figma-style-binding |
| Find components, |     |  Bind colors/fonts/  |
| build structure  |     |  spacing to tokens   |
+--------+--------+     +----------+----------+
         |                        |
         v                        v
    +-------------------------------------+
    |       figma-qa-verifier             |
    |  Verify Style/Variable bindings     |
    +-------------------------------------+
```

---

## Supported Scenarios

| Scenario | How It Works |
|----------|-------------|
| **Working in a DS file directly** | Preflight reads all local Styles, Variables, and Components. Full token binding. |
| **New file with a linked DS library** | Instances imported via `importComponentByKeyAsync` inherit all token bindings from the master component automatically. Library variables can be imported via `importVariableByKeyAsync` for new frames. |

---

## Directory Structure

```
your-project/
├── CLAUDE.md                          # Project config (Figma URL, fonts, rules)
└── .claude/
    ├── settings.json                  # Permissions + QA Hook
    └── skills/
        ├── figma-preflight/
        │   └── SKILL.md               # 7 checks + Token Map + Component Registry
        ├── component-rules/
        │   └── SKILL.md               # Component build rules (Library-first, Auto Layout, naming)
        ├── figma-style-binding/
        │   └── SKILL.md               # Style binding rules (Color / Text / Spacing)
        ├── figma-qa-verifier/
        │   └── SKILL.md               # QA verification rules
        └── reference-interpreter/
            └── SKILL.md               # Reference → Design Brief
```

---

## Core Principles

1. **Components first** — Search the Design System before building from scratch
2. **Token-bound** — All visual values must bind to a Variable or Style, no hardcoded values
3. **Incremental** — Build one section at a time, verify with screenshot before continuing
4. **Semantic naming** — Every node uses `"Section / Role"` format
5. **Transparent QA** — Every operation returns node IDs, bindings are verified automatically

## Known Limitations

| Limitation | Details | Workaround |
|------------|---------|------------|
| Nested instance modification | Doubly-nested Instances can't be directly modified | `detachInstance()` on the outer instance first |
| Library variables not enumerable | `getLocalVariablesAsync()` can't see library variables | Instances inherit tokens automatically; use `importVariableByKeyAsync` for new frames |
| Variant property values | Invalid values in `setProperties` roll back the entire atomic script | Read `componentPropertyDefinitions` to confirm valid values first |

---

## Contributing

Issues and PRs are welcome. If you have a Figma design workflow that could benefit from a new Skill, please describe it in an Issue.

## License

MIT © 2025 Sen Lin
