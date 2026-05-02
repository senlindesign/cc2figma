---
name: figma-preflight
description: "Triggers on 'let's start', 'begin', 'preflight', 'start the session', or when a Figma file URL is first shared. Verifies MCP connection, reads CLAUDE.md, audits connected libraries, and loads a Token Map of all Styles and Variables — required before any design work."
disable-model-invocation: false
---

# Figma Preflight

Run at the start of every design session. Do NOT start design work until all steps pass.

**Prerequisite:** Load `figma-use` skill before any `use_figma` call.

---

## Step A — Connection + Config (parallel)

Run these two in parallel:

1. **MCP Connection:** Call `mcp__figma__whoami`. Must return user email and plan. Fail → stop, re-authenticate.
2. **CLAUDE.md:** Read CLAUDE.md. Extract Figma file URL (required — stop if missing), font families, session goal. If fonts field is a placeholder, auto-populate after Step C using STRING variables starting with "Family".

---

## Step B — File + Libraries (parallel)

Parse `fileKey` from the Figma URL, then run in parallel:

1. **File Access:** Call `get_metadata` with extracted nodeId and fileKey. Must return file name and pages.
2. **Libraries:** Call `get_libraries` with fileKey. Store subscribed libraries as **Library Registry** (name, libraryKey, description). These enable `search_design_system` to find library styles and components during design work.

---

## Step C — Styles + Variables + Components (single use_figma call)

Combine all three inventories in one script:

```javascript
const textStyles = await figma.getLocalTextStylesAsync();
const paintStyles = await figma.getLocalPaintStylesAsync();
const collections = await figma.variables.getLocalVariableCollectionsAsync();
const variables = await figma.variables.getLocalVariablesAsync();

const grouped = {};
for (const v of variables) {
  const key = v.resolvedType;
  if (!grouped[key]) grouped[key] = [];
  grouped[key].push({ name: v.name, scopes: v.scopes });
}

const components = {};
for (const page of figma.root.children) {
  await figma.setCurrentPageAsync(page);
  const sets = page.findAll(n => n.type === "COMPONENT_SET");
  const solos = page.findAll(n => n.type === "COMPONENT" && n.parent.type !== "COMPONENT_SET");
  if (sets.length > 0 || solos.length > 0) {
    components[page.name] = {
      sets: sets.map(c => c.name),
      solos: solos.map(c => c.name).slice(0, 15),
    };
  }
}

return {
  textStyles: textStyles.map(s => s.name),
  paintStyles: paintStyles.map(s => s.name),
  collections: collections.map(c => c.name),
  variableCount: variables.length,
  byType: Object.fromEntries(
    Object.entries(grouped).map(([type, vars]) => [type, vars.map(v => v.name)])
  ),
  components
};
```

Store **names only** in context. IDs are looked up on-demand during design work. Library styles/variables are discovered via `search_design_system`.

---

## Token Map

After Step C, derive a semantic index from variables grouped by `scopes`:

| Role | Scope | Example names |
|---|---|---|
| Background fill | `FRAME_FILL`, `SHAPE_FILL` | `background/surface`, `color/neutral-100` |
| Text color | `TEXT_FILL` | `text/primary`, `color/neutral-900` |
| Border / stroke | `STROKE_COLOR` | `border/default`, `color/neutral-300` |
| Gap | `GAP` | `gap/sm`, `spacing/xxs` |
| Padding | `PADDING` | `padding/md`, `spacing/section-xl` |
| Border radius | `CORNER_RADIUS` | `radius/sm`, `radius/full` |

---

## Status Report

```
✅ MCP Connection    — [name] ([email]) · [plan]
✅ CLAUDE.md         — Font: [primary] / [code] · Goal: [goal]
✅ Figma File        — [file name] · [N] pages
✅ Libraries         — [N] connected: [names]
✅ Styles            — [N] text + [N] paint
✅ Variables         — [N] across [N] collections
✅ Components        — [N] sets across [N] pages

── Token Map ──────────────────────────────
Background  : [names]
Text        : [names]
Border      : [names]
Gap         : [names]
Padding     : [names]
Radius      : [names]
────────────────────────────────────────────

Ready to design.
```

If any step fails, output ❌ with error and stop.
