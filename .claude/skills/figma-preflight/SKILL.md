---
name: figma-preflight
description: "Triggers on 'let's start', 'begin', 'preflight', 'start the session', or when a Figma file URL is first shared. Verifies MCP connection, reads CLAUDE.md, audits connected libraries, and loads a Token Map of all Styles and Variables — required before any design work."
disable-model-invocation: false
---

# Figma Preflight Skill

Run this at the beginning of every design session. The goal: verify everything is connected and load the complete Style, Variable, and Library inventory from the Figma file into context, so all subsequent design work binds properly to the design system.

Do NOT start any design work until all 7 checks pass.

**Prerequisite:** Before running Checks 4 and 5, load the `figma-use` skill. It is a mandatory prerequisite for every `use_figma` call.

---

## Check 1 — MCP Connection

Call `mcp__figma__whoami` to verify the Figma MCP server is authenticated and reachable.

**Pass condition:** Returns user email, handle, and plan name.
**Fail action:** Stop. Ask the user to re-authenticate via the Figma MCP server before continuing.

---

## Check 2 — CLAUDE.md

Read the CLAUDE.md file from the current project directory.

Extract and confirm these fields are present:
- **Figma file URL** — required. Stop if missing; ask the user to add it.
- **Font families** — if the field contains a placeholder (e.g. `[primary / headings]`), auto-populate after Check 6 completes (see below).
- **Session goal**

**Font auto-population (run after Check 6 if Fonts field is a placeholder):** Read STRING variables from the Variable Registry and update CLAUDE.md automatically:

```javascript
const variables = await figma.variables.getLocalVariablesAsync();
const collections = await figma.variables.getLocalVariableCollectionsAsync();
const collMap = Object.fromEntries(collections.map(c => [c.id, c]));
const fontVars = variables.filter(v => v.resolvedType === "STRING" && v.name.startsWith("Family"));
return fontVars.map(v => ({
  name: v.name,
  value: v.valuesByMode[collMap[v.variableCollectionId].defaultModeId]
}));
```

Map the results to CLAUDE.md: `Family Sans` → primary font, `Family Mono` → code font, `Family Serif` → serif font (if used). Edit CLAUDE.md directly with the resolved values.

---

## Check 3 — Figma File Access

Parse the `fileKey` from the Figma URL found in CLAUDE.md.

Call `get_metadata` with `nodeId: "0:1"` and the extracted `fileKey`. Verify the file is accessible.

**Pass condition:** Returns file name and page list.
**Fail action:** Stop. Report the specific access error (permissions, wrong key, network).

---

## Check 4 — Connected Libraries (Library Registry)

Call `mcp__figma__get_libraries` with the `fileKey` from CLAUDE.md.

This returns two lists:
1. **Subscribed libraries** — currently connected to this file (these are the usable ones)
2. **Available libraries** — organization/community libs available to add

Store the subscribed libraries as the **Library Registry** for this session. Record each library's `name`, `libraryKey`, and `description`.

**Why this matters:** In professional Figma workflows, Text Styles, Color Styles, and Components almost always come from external published libraries — not from the local file. `getLocalTextStylesAsync()` cannot see these. The Library Registry is what enables `search_design_system` to find and bind to the real design system tokens later.

**Pass condition:** Returns at least one subscribed library, OR the file has no connected libraries (record as empty, continue).

---

## Check 5 — Local Style Inventory (Style Registry)

> Load `figma-use` skill before running this check.

Run the following script via `use_figma` to count and list **locally defined** Text Styles and Paint Styles:

```javascript
const textStyles = await figma.getLocalTextStylesAsync();
const paintStyles = await figma.getLocalPaintStylesAsync();

return {
  textStyleCount: textStyles.length,
  textStyleNames: textStyles.map(s => s.name),
  paintStyleCount: paintStyles.length,
  paintStyleNames: paintStyles.map(s => s.name)
};
```

Store the **names only** in context (the Token Map). Do NOT store full objects with IDs — IDs are looked up on-demand during design work (see "On-Demand ID Lookup" below).

**Important limitation:** This only returns styles *defined inside this file*. Library styles must be discovered via `search_design_system` during design work.

---

## Check 6 — Local Variable Inventory (Variable Registry)

> Load `figma-use` skill before running this check.

Run the following script via `use_figma` to count and list **locally defined** Variables, grouped by scope:

```javascript
const collections = await figma.variables.getLocalVariableCollectionsAsync();
const variables = await figma.variables.getLocalVariablesAsync();

const grouped = {};
for (const v of variables) {
  const key = v.resolvedType;
  if (!grouped[key]) grouped[key] = [];
  grouped[key].push({ name: v.name, scopes: v.scopes });
}

return {
  collectionCount: collections.length,
  collectionNames: collections.map(c => c.name),
  variableCount: variables.length,
  byType: Object.fromEntries(
    Object.entries(grouped).map(([type, vars]) => [type, vars.map(v => v.name)])
  )
};
```

Store the **names grouped by type** in context (feeds the Token Map). Do NOT store full objects with IDs — IDs are looked up on-demand during design work (see "On-Demand ID Lookup" below).

**Same limitation applies:** Library variables must be discovered via `search_design_system` during design work.

---

## Check 7 — Local Component Inventory (Component Registry)

> Load `figma-use` skill before running this check.

Scan each page individually for COMPONENT_SET and solo COMPONENT nodes. Do **not** use `figma.root.findAll()` — it scans all pages at once and times out on large files.

```javascript
const results = {};
for (const page of figma.root.children) {
  await figma.setCurrentPageAsync(page);
  const sets = page.findAll(n => n.type === "COMPONENT_SET");
  const solos = page.findAll(
    n => n.type === "COMPONENT" && n.parent.type !== "COMPONENT_SET"
  );
  if (sets.length > 0 || solos.length > 0) {
    results[page.name] = {
      sets: sets.map(c => c.name),
      solos: solos.map(c => c.name).slice(0, 15), // cap icon-heavy pages
    };
  }
}
return results;
```

Store the component names grouped by page as the **Component Registry**. This tells Claude what's available for `createInstance()` without needing a separate search each time.

**Pass condition:** Scan completes. Zero local components is valid — record as empty and continue.

**Why this matters:** Without a Component Registry, every design task requires a manual scan. With it, Claude immediately knows whether `Form Log In`, `Button`, `Tab`, etc. already exist locally — enabling the library-first lookup from `component-rules` without extra round trips.

---

## Token Map (generated from Variable Registry)

After storing the Variable Registry, immediately derive the **Token Map** — a semantic index that tells Claude which variable to use for each design decision. This replaces the need for a separate design-tokens skill.

Group variables by their `scopes` field and name patterns:

| Semantic role | Scope to match | Example variable names |
|---|---|---|
| Background fill | `FRAME_FILL`, `SHAPE_FILL` | `color/neutral-100`, `background/surface` |
| Text color | `TEXT_FILL` | `color/neutral-900`, `text/primary` |
| Border / stroke | `STROKE_COLOR` | `color/neutral-300`, `border/default` |
| Gap between items | `GAP` | `gap/sm`, `spacing/xxs` |
| Padding (container) | `PADDING` | `padding/md`, `spacing/section-xl` |
| Border radius | `CORNER_RADIUS` | `border/radius-sm`, `radius/full` |
| Width / height | `WIDTH_HEIGHT` | `size/icon-md`, `width/button-lg` |

Output the Token Map as a compact table after the Status Report. During design work, consult the Token Map first when deciding which variable to bind — it is faster than scanning the full Variable Registry.

---

## Session Status Report

After all 7 checks, output a formatted status block followed by the Token Map:

```
✅ MCP Connection    — Authenticated as [name] ([email]) · [plan]
✅ CLAUDE.md         — Loaded from [path]
                       Font: [primary] / [code]
                       Goal: [session goal]
✅ Figma File        — [file name] · [N] pages
✅ Libraries         — [N] connected: [lib name 1], [lib name 2], ...
✅ Text Styles       — [N] local styles in Style Registry
✅ Paint Styles      — [N] local styles in Style Registry
✅ Variables         — [N] variables across [N] collections in Variable Registry
✅ Components        — [N] component sets across [N] pages
                       [Page]: [CompA] · [CompB] · [CompC]
                       [Page]: [CompA] · [CompB]

── Token Map ──────────────────────────────────────────
Background fills  : [var name] · [var name] · ...
Text colors       : [var name] · [var name] · ...
Border / stroke   : [var name] · ...
Gap               : [var name] · [var name] · ...
Padding           : [var name] · [var name] · ...
Border radius     : [var name] · ...
────────────────────────────────────────────────────────

Note: Library styles/components are discoverable via search_design_system during design work.

Ready to design. Share a reference or describe the section to build.
```

If any check fails, output ❌ with the specific error and stop. Do not proceed to design.

---

## Session Rules (Active After Preflight Passes)

Once preflight passes, the Token Map and Library Registry are active. These rules apply to every `use_figma` call for the rest of the session:

### Before creating anything — search first

For any component, style, or token, always search the connected libraries before creating from scratch:

```
search_design_system(query: "[what you need]", fileKey: "[fileKey]", includeLibraryKeys: [libraryKeys from Library Registry])
```

If a matching component or style is found in a library, import and use it. Never recreate what already exists in the design system.

### Text nodes

Look up the matching Text Style in the Style Registry by name. Apply via:
```javascript
node.textStyleId = "<id from Style Registry>";
```

If the style is not in the local Style Registry, search the connected libraries via `search_design_system` and import the style. Never set `fontSize`, `fontFamily`, `fontWeight`, or `lineHeight` as raw values.

### Color fills

Look up the matching Paint Style or COLOR Variable in the registries. Apply via:
```javascript
// Option A — Paint Style
node.fillStyleId = "<id from Style Registry>";

// Option B — Variable (preferred for tokens/themes)
const variable = await figma.variables.getVariableByIdAsync("<id from Variable Registry>");
const fill = { type: "SOLID", color: { r: 0, g: 0, b: 0 } };
node.fills = [figma.variables.setBoundVariableForPaint(fill, "color", variable)];
```

Never use raw hex values or `{ r, g, b }` objects without a Variable or Style binding.

### Spacing, padding, gap, border radius

Look up the closest matching FLOAT Variable in the Variable Registry. Apply via:
```javascript
node.setBoundVariable("paddingLeft", variable);
node.setBoundVariable("paddingRight", variable);
node.setBoundVariable("paddingTop", variable);
node.setBoundVariable("paddingBottom", variable);
node.setBoundVariable("itemSpacing", variable);
node.setBoundVariable("cornerRadius", variable);
```

Prefer Variable binding over raw numbers. If no matching Variable exists, report the gap — do not silently use a raw value.

### No matching style or variable found

Surface the gap and wait for user decision:
```
⚠️ No match found for "[semantic need]" in Style Registry or Variable Registry.
   Closest available: "[name]" — proceed with this, or add the missing token first?
```

Do not proceed without the user's confirmation. Do not silently fall back to a raw value.

### Spot-checking applied styles during QA

To verify what styles and variables are applied to specific nodes after design work, use the `get_variable_defs` MCP tool on the node's selection. This is faster than running a full Plugin API script and is useful for lightweight QA checks.

---

## On-Demand ID Lookup

Preflight stores only **names** to keep context lean. When you need a specific Style or Variable ID during design work, run a lightweight lookup:

### Look up a Text Style ID by name
```javascript
const styles = await figma.getLocalTextStylesAsync();
const match = styles.find(s => s.name === "heading/h1");
return match ? { id: match.id, name: match.name } : { error: "Not found" };
```

### Look up a Paint Style ID by name
```javascript
const styles = await figma.getLocalPaintStylesAsync();
const match = styles.find(s => s.name === "surface/primary");
return match ? { id: match.id, name: match.name } : { error: "Not found" };
```

### Look up a Variable ID by name
```javascript
const variables = await figma.variables.getLocalVariablesAsync();
const match = variables.find(v => v.name === "color/primary-500");
return match ? { id: match.id, name: match.name, resolvedType: match.resolvedType } : { error: "Not found" };
```

These lookups are fast (single API call each) and avoid storing hundreds of IDs in session context. Use the Token Map names from preflight to know WHAT to look up, then use these scripts to get the actual ID right before applying it.
