---
name: component-rules
description: "Triggers when building any UI element in Figma — 'create a card', 'build a nav', 'add a section', 'make a component'. Enforces library-first component lookup, correct Auto Layout structure, and semantic node naming. For visual property binding (colors, text styles, spacing values), defer to figma-style-binding."
disable-model-invocation: false
---

# component-rules

This skill governs how Claude constructs UI components in Figma. It enforces a strict library-first principle and correct Auto Layout usage. The goal: Claude never rebuilds things that already exist in the connected design system, and everything it builds follows proper structure.

**Integration with official skills:** This skill supplements Figma's built-in `figma-generate-design` skill. When both are loaded, structural rules from this skill (library lookup, Auto Layout setup, naming) apply to every frame that `figma-generate-design` creates. For visual property binding (colors, text styles, spacing variable values), defer to `figma-style-binding`.

---

## Trigger conditions

Load this skill when the user says things like:
- "build a component", "create a card", "add a nav bar", "build this section"
- "make a button row", "add a form", "build this layout"
- Any construction task that involves creating nodes in Figma
- Works alongside `figma-style-binding` (visual properties) and `figma-preflight` (session setup)

---

## Rule 1 — Library-first hierarchy

Before writing any construction code, always resolve the component source in this order:

```
1. search_design_system → found? → importComponentByKeyAsync → createInstance()
2. figma.currentPage.findAll(n => n.type === "COMPONENT") → found? → createInstance()
3. Build from scratch only if nothing matches
```

Never rebuild primitives that design systems already provide:

| Never rebuild | Examples |
|---|---|
| Interactive controls | Button, Input, Checkbox, Radio, Toggle, Select |
| Display atoms | Badge, Tag, Chip, Avatar, Icon |
| Navigation | Tab, Breadcrumb, Link Item, Menu Item |
| Feedback | Toast, Alert, Spinner, Progress |

### Decision tree (pseudocode)

```js
async function resolveComponent(query) {
  // Step 1: Search connected libraries
  const results = await searchDesignSystem(query); // via search_design_system MCP tool
  if (results.length > 0) {
    const comp = await figma.importComponentByKeyAsync(results[0].key);
    return comp.createInstance();
  }

  // Step 2: Check local components in the file
  const local = figma.currentPage.findAll(
    n => n.type === "COMPONENT" && n.name.toLowerCase().includes(query.toLowerCase())
  );
  if (local.length > 0) {
    return local[0].createInstance();
  }

  // Step 3: Build from scratch — only reached if nothing exists
  return buildFromScratch(query);
}
```

### Importing from a library

```js
// Always prefer importComponentByKeyAsync for library components
const buttonComp = await figma.importComponentByKeyAsync("abc123def456"); // key from search_design_system
const buttonInstance = buttonComp.createInstance();
parent.appendChild(buttonInstance);

// For component sets (variants), import the set then find the right variant
const compSet = await figma.importComponentSetByKeyAsync("xyz789");
const primaryVariant = compSet.defaultVariant; // or filter by variantProperties
const instance = primaryVariant.createInstance();
parent.appendChild(instance);
```

---

## Rule 2 — Auto Layout setup

Every frame container must use Auto Layout. Never leave `layoutMode` as `"NONE"` for structural containers.

### Required property sequence

Order matters. The Plugin API requires `layoutMode` to be set before layout-related properties.

```js
const frame = figma.createFrame();
frame.name = "Card / Container";

// 1. Set layout mode FIRST
frame.layoutMode = "VERTICAL";           // or "HORIZONTAL"

// 2. Set sizing modes
frame.primaryAxisSizingMode = "AUTO";    // hug content vertically
frame.counterAxisSizingMode = "FIXED";  // fixed width

// 3. Set sizing behavior AFTER appending to parent
parent.appendChild(frame);
frame.layoutSizingHorizontal = "FILL";  // ← must come AFTER append
frame.layoutSizingVertical = "HUG";

// 4. Set spacing (raw numbers only if no variable — prefer Rule 3 binding)
frame.paddingTop = 16;
frame.paddingBottom = 16;
frame.paddingLeft = 24;
frame.paddingRight = 24;
frame.itemSpacing = 12;

// 5. Alignment
frame.primaryAxisAlignItems = "MIN";     // "MIN" | "MAX" | "CENTER" | "SPACE_BETWEEN"
frame.counterAxisAlignItems = "MIN";     // "MIN" | "MAX" | "CENTER" | "BASELINE"
```

### Sizing cheat sheet

| Goal | primaryAxisSizingMode | counterAxisSizingMode | layoutSizingHorizontal | layoutSizingVertical |
|---|---|---|---|---|
| Hug content (both axes) | AUTO | AUTO | HUG | HUG |
| Fixed card (e.g. 360x200) | FIXED | FIXED | FIXED | FIXED |
| Full-width section | AUTO | FIXED | FILL | HUG |
| Stretch in parent | AUTO | AUTO | FILL | FILL |

### FILL parent width — default for section-level frames

```js
// Pattern: full-width section with vertical stack
const section = figma.createFrame();
section.name = "Hero / Section";
section.layoutMode = "VERTICAL";
section.primaryAxisSizingMode = "AUTO";
section.counterAxisSizingMode = "AUTO";
page.appendChild(section);
section.layoutSizingHorizontal = "FILL"; // AFTER append
section.layoutSizingVertical = "HUG";
```

---

## Rule 3 — Visual property binding (defer to figma-style-binding)

All color fills, text styles, spacing variable bindings, and corner radius bindings are governed by `figma-style-binding`. Refer to that skill for lookup, application, and gap-handling rules.

**Structural prerequisite from this skill:** `layoutMode` must be set BEFORE any `setBoundVariable` call. The Plugin API will throw otherwise.

```js
// CORRECT — layoutMode before setBoundVariable
frame.layoutMode = "VERTICAL";
frame.setBoundVariable("paddingTop", spacingVar);

// WRONG — will throw
frame.setBoundVariable("paddingTop", spacingVar); // layoutMode not set yet
frame.layoutMode = "VERTICAL";
```

---

## Rule 4 — Node naming conventions

Name every node semantically using the Figma slash-hierarchy convention. Never leave default names.

### Format: `"Section / Role"`

```
Component / Container
Component / Header
Component / Title
Component / Subtitle
Component / Body
Component / Footer
Component / Action Row
```

### Good vs bad examples

| Bad (default or generic) | Good (semantic) |
|---|---|
| Frame 12 | Card / Container |
| Rectangle 5 | Hero / Background |
| Group | Nav / Link Group |
| Text | Card / Title |
| Instance | Button / Primary |
| Ellipse | Avatar / Image |

### Apply naming at creation time

```js
const card = figma.createFrame();
card.name = "Card / Container";

const title = figma.createText();
title.name = "Card / Title";

const body = figma.createText();
body.name = "Card / Body";

const actions = figma.createFrame();
actions.name = "Card / Action Row";
```

---

## Rule 5 — Return all node IDs

Every `use_figma` call must return an object with all created and mutated node IDs for QA verification.

```js
// At the end of every script, return IDs
return {
  card: card.id,
  title: title.id,
  body: body.id,
  actionRow: actionRow.id,
  buttonInstance: buttonInstance.id,
};
```

If you mutated existing nodes, include them too:

```js
return {
  created: {
    wrapper: wrapper.id,
    heading: heading.id,
  },
  mutated: {
    existingSection: existingSection.id,
  },
};
```

---

## Rule 6 — Incremental building

Build one component or section at a time. Validate via screenshot or node read after each step. Do not create entire pages in a single script.

### Recommended sequence for a composed view

```
Step 1: Build the outermost container frame → validate layout
Step 2: Add header/nav section → validate
Step 3: Add main content area → validate
Step 4: Add each card or list item → validate each
Step 5: Add footer / action area → validate
```

Each step = one `use_figma` call with its own return of node IDs.

---

## Full example — Card component

```js
// ── Card Component — full example ──────────────────────────────────────────

// Step 1: Resolve avatar from library
const avatarComp = await figma.importComponentByKeyAsync("AVATAR_KEY_FROM_SEARCH");
const avatar = avatarComp.createInstance();

// Step 2: Resolve button from library
const buttonComp = await figma.importComponentByKeyAsync("BUTTON_KEY_FROM_SEARCH");
const ctaButton = buttonComp.createInstance();

// Step 3: Get spacing variables
const allVars = await figma.variables.getLocalVariablesAsync();
const spacingMd = allVars.find(v => v.name === "spacing/md");
const spacingLg = allVars.find(v => v.name === "spacing/lg");
const radiusMd = allVars.find(v => v.name === "radius/md");

// Step 4: Build the card container
const card = figma.createFrame();
card.name = "Card / Container";
card.layoutMode = "VERTICAL";
card.primaryAxisSizingMode = "AUTO";
card.counterAxisSizingMode = "FIXED";
card.resize(360, card.height); // set width, let height hug
figma.currentPage.appendChild(card);
card.layoutSizingHorizontal = "HUG"; // AFTER append

// Bind spacing variables
if (spacingMd) {
  card.setBoundVariable("paddingTop", spacingMd);
  card.setBoundVariable("paddingBottom", spacingMd);
  card.setBoundVariable("paddingLeft", spacingLg || spacingMd);
  card.setBoundVariable("paddingRight", spacingLg || spacingMd);
  card.setBoundVariable("itemSpacing", spacingMd);
}
if (radiusMd) {
  card.setBoundVariable("cornerRadius", radiusMd);
}

// Step 5: Header row (avatar + name stack)
const header = figma.createFrame();
header.name = "Card / Header";
header.layoutMode = "HORIZONTAL";
header.primaryAxisSizingMode = "AUTO";
header.counterAxisSizingMode = "AUTO";
card.appendChild(header);
header.layoutSizingHorizontal = "FILL";
if (spacingMd) header.setBoundVariable("itemSpacing", spacingMd);

header.appendChild(avatar);

const nameStack = figma.createFrame();
nameStack.name = "Card / Name Stack";
nameStack.layoutMode = "VERTICAL";
nameStack.primaryAxisSizingMode = "AUTO";
nameStack.counterAxisSizingMode = "AUTO";
header.appendChild(nameStack);
nameStack.layoutSizingHorizontal = "FILL";

// Step 6: Text nodes
await figma.loadFontAsync({ family: "Inter", style: "Semi Bold" });
await figma.loadFontAsync({ family: "Inter", style: "Regular" });

const nameText = figma.createText();
nameText.name = "Card / Name";
nameText.fontName = { family: "Inter", style: "Semi Bold" };
nameText.characters = "Alex Johnson";
nameStack.appendChild(nameText);

const roleText = figma.createText();
roleText.name = "Card / Role";
roleText.fontName = { family: "Inter", style: "Regular" };
roleText.characters = "Product Designer";
nameStack.appendChild(roleText);

// Step 7: Body text
const bodyText = figma.createText();
bodyText.name = "Card / Body";
bodyText.fontName = { family: "Inter", style: "Regular" };
bodyText.characters = "Building design systems that scale across teams.";
card.appendChild(bodyText);
bodyText.layoutSizingHorizontal = "FILL";

// Step 8: Action row
const actionRow = figma.createFrame();
actionRow.name = "Card / Action Row";
actionRow.layoutMode = "HORIZONTAL";
actionRow.primaryAxisSizingMode = "AUTO";
actionRow.counterAxisSizingMode = "AUTO";
actionRow.primaryAxisAlignItems = "MAX"; // right-align
card.appendChild(actionRow);
actionRow.layoutSizingHorizontal = "FILL";

actionRow.appendChild(ctaButton);

// Step 9: Return all IDs
return {
  card: card.id,
  header: header.id,
  avatar: avatar.id,
  nameStack: nameStack.id,
  nameText: nameText.id,
  roleText: roleText.id,
  bodyText: bodyText.id,
  actionRow: actionRow.id,
  ctaButton: ctaButton.id,
};
```

---

## Companion skills

| Skill | Role |
|---|---|
| `figma-preflight` | Run first — loads Style Registry, Variable Registry, Library Registry |
| `figma-style-binding` | Governs fill color, text style, stroke binding |
| `component-rules` (this skill) | Governs structure, layout, naming, library lookup |
| `figma-use` | Mandatory prerequisite before every `use_figma` tool call |

---

## Quick-reference checklist

Before submitting any construction script, verify:

- [ ] `search_design_system` was called for every primitive before building from scratch
- [ ] Every frame container has `layoutMode` set to `"VERTICAL"` or `"HORIZONTAL"`
- [ ] `layoutSizingHorizontal` / `layoutSizingVertical` set AFTER `appendChild`
- [ ] `layoutMode` set BEFORE any `setBoundVariable` call
- [ ] All spacing uses variable binding (if variables exist in the file)
- [ ] Every node has a semantic name using "/" hierarchy
- [ ] Script returns all created and mutated node IDs
- [ ] Script builds one section at a time, not the entire page
