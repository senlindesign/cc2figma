---
name: figma-style-binding
description: "Triggers on any visual property change in Figma — creating text, setting colors, adjusting spacing/padding/gap/radius. Enforces that ALL values bind to Figma Styles or Variables, never hardcoded. Works alongside figma-generate-design for screen building."
disable-model-invocation: false
---

# Figma Style Binding Skill

This skill governs how Claude sets every visual property in Figma. The core rule is simple: **every value must come from the design system, not from Claude's own judgment.**

**Integration with official skills:** This skill supplements Figma's built-in `figma-generate-design` skill. When both are loaded, binding rules from this skill take precedence — every value that `figma-generate-design` would place must go through the binding hierarchy defined here. Load both skills together for screen-level design work.

Prerequisite: `figma-preflight` must have already run this session, so the Token Map and Library Registry are available in context.

---

## The Binding Hierarchy

When setting any visual property, always follow this order. Stop at the first match.

```
1. Connected Library  →  search_design_system → import → apply
2. Local Style        →  Style Registry → apply by ID
3. Local Variable     →  Variable Registry → apply by ID
4. Gap found          →  Report to user, wait for decision
```

Never skip to step 4 without genuinely exhausting steps 1–3.

---

## Text Styling

### Rule
Every text node must have its `textStyleId` set. Setting individual font properties (`fontSize`, `fontFamily`, `fontWeight`, `lineHeight`, `letterSpacing`) directly is forbidden unless explicitly told there are no text styles in this file.

### Lookup process
1. Look at the Style Registry from preflight. Find the style whose name best matches the semantic role (e.g. "heading/h1", "body/regular", "label/small").
2. If not in the local Style Registry, search connected libraries:
   ```
   search_design_system(query: "heading h1", fileKey: "...", includeLibraryKeys: [...])
   ```
3. Import the matching style from the library if found.

### Apply
```javascript
// Load the font before setting style — required by Plugin API
const style = await figma.getStyleByIdAsync("<id from Style Registry>");
await figma.loadFontAsync(style.fontName);

const node = figma.createText();
node.textStyleId = "<id from Style Registry>";
node.characters = "Your text here";
```

### When no text style matches
```
⚠️ Text style gap: no style found for "[semantic role]".
   Available styles: [list top 5 from Style Registry]
   Options: (a) use closest match "[name]", (b) add missing style first
```
Wait for user confirmation before proceeding.

---

## Color Fills

### Rule
Every fill or stroke must be bound to either a Paint Style (`fillStyleId`) or a COLOR Variable (`setBoundVariableForPaint`). Raw `{ r, g, b }` objects without a binding are forbidden.

Variables are preferred over Paint Styles when both exist, because Variables support theming (light/dark modes).

### Lookup process
1. Check Variable Registry for a COLOR variable matching the semantic need (e.g. "color/primary-500", "background/surface").
2. If no Variable match, check Style Registry for a Paint Style.
3. If neither, search connected libraries via `search_design_system`.

### Apply — Variable binding (preferred)
```javascript
// setBoundVariableForPaint returns a NEW paint object — must reassign
const variable = await figma.variables.getVariableByIdAsync("<id from Variable Registry>");
const baseFill = { type: "SOLID", color: { r: 0, g: 0, b: 0 } };
const boundFill = figma.variables.setBoundVariableForPaint(baseFill, "color", variable);
node.fills = [boundFill];
```

### Apply — Paint Style binding
```javascript
node.fillStyleId = "<id from Style Registry>";
```

### Apply — Stroke binding
```javascript
node.strokeStyleId = "<id from Style Registry>";
// or for variable:
const boundStroke = figma.variables.setBoundVariableForPaint(baseStroke, "color", variable);
node.strokes = [boundStroke];
```

### When no color match found
```
⚠️ Color gap: no style or variable found for "[semantic need]" (e.g. primary background).
   Closest in registry: "[name]" — proceed with this, or add the missing token first?
```

---

## Spacing, Padding, and Gap

### Rule
All Auto Layout spacing properties must be bound to FLOAT Variables from the Variable Registry. Never write raw pixel numbers for padding, gap, or border radius.

### Lookup process
Check the Variable Registry for FLOAT variables with scopes including `GAP`, `PADDING`, or `CORNER_RADIUS`. Match by semantic name (e.g. "spacing/sm", "gap/xxs", "padding/section-xl").

### Apply
```javascript
const spacingVar = await figma.variables.getVariableByIdAsync("<id from Variable Registry>");

// Padding
node.setBoundVariable("paddingTop", spacingVar);
node.setBoundVariable("paddingBottom", spacingVar);
node.setBoundVariable("paddingLeft", spacingVar);
node.setBoundVariable("paddingRight", spacingVar);

// Gap between children
node.setBoundVariable("itemSpacing", spacingVar);

// Border radius
node.setBoundVariable("cornerRadius", spacingVar);
```

Note: `setBoundVariable` only works on Auto Layout frames. Set `node.layoutMode = "VERTICAL"` or `"HORIZONTAL"` before calling it.

### When no spacing variable matches
```
⚠️ Spacing gap: no variable found for "[semantic need]" (e.g. section padding top).
   Available FLOAT variables: [list matching-scope vars from Variable Registry]
   Options: (a) use closest match "[name]", (b) proceed with raw value [N]px temporarily
```
Unlike color and text, spacing can fall back to a raw value temporarily with user confirmation — a raw spacing value does not break the design system connection the way a raw color or text style does.

---

## Components

### Rule
Before creating any UI element from scratch, check if it already exists as a component in the connected libraries.

```
search_design_system(query: "[component name]", fileKey: "...", includeLibraryKeys: [...])
```

If a match is found, import and use it:
```javascript
const component = await figma.importComponentByKeyAsync("<componentKey>");
const instance = component.createInstance();
```

Only build from scratch if no match exists in any connected library. Never recreate primitives (buttons, inputs, badges, icons) that the design system already provides.

---

## What Claude Must Never Do

| ❌ Forbidden | ✅ Required instead |
|---|---|
| `node.fontSize = 24` | `node.textStyleId = id` |
| `node.fills = [{ type: "SOLID", color: { r: 0.2, g: 0.4, b: 1 } }]` | Variable or Style binding |
| `node.paddingLeft = 16` | `node.setBoundVariable("paddingLeft", var)` |
| `node.cornerRadius = 8` | `node.setBoundVariable("cornerRadius", var)` |
| Creating a Button component from scratch | `importComponentByKeyAsync` from library |

---

## After Creating Nodes — Always Return IDs

Every `use_figma` call that creates or modifies nodes must return the affected node IDs. This is required for QA verification and follow-up edits.

```javascript
return {
  createdNodeIds: [frame.id, textNode.id, ...],
  mutatedNodeIds: [existingNode.id, ...]
};
```
