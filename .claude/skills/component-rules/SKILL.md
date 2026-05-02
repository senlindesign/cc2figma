---
name: component-rules
description: "Triggers when building any UI element in Figma — 'create a card', 'build a nav', 'add a section', 'make a component'. Enforces library-first component lookup, correct Auto Layout structure, and semantic node naming. For visual property binding (colors, text styles, spacing values), defer to figma-style-binding."
disable-model-invocation: false
---

# Component Rules

Governs how Claude constructs UI in Figma. Supplements `figma-generate-design`. For visual property binding, defer to `figma-style-binding`.

---

## Rule 1 — Library-first hierarchy

Before building anything, resolve the component source in this order:

```
1. search_design_system → importComponentByKeyAsync → createInstance()
2. Local file scan → createInstance()
3. Build from scratch — ONLY if nothing matches
```

Never rebuild primitives the DS provides: Button, Input, Checkbox, Toggle, Badge, Tag, Avatar, Icon, Tab, Breadcrumb, Toast, Alert, Spinner.

```js
// Library import
const comp = await figma.importComponentByKeyAsync("key_from_search");
const instance = comp.createInstance();
parent.appendChild(instance);

// For component sets (variants)
const set = await figma.importComponentSetByKeyAsync("key");
const instance = set.defaultVariant.createInstance();
```

---

## Rule 2 — Auto Layout

Every container must use Auto Layout. Property order matters:

1. Set `layoutMode` FIRST (`"VERTICAL"` or `"HORIZONTAL"`)
2. Set `layoutSizingHorizontal` / `layoutSizingVertical` AFTER `appendChild`
3. Set `layoutMode` BEFORE any `setBoundVariable` call

| Goal | primaryAxisSizing | counterAxisSizing | layoutSizingH | layoutSizingV |
|---|---|---|---|---|
| Hug both | AUTO | AUTO | HUG | HUG |
| Fixed card | FIXED | FIXED | FIXED | FIXED |
| Full-width section | AUTO | FIXED | FILL | HUG |

---

## Rule 3 — Node naming

Name every node semantically with slash hierarchy. Never leave defaults.

```
Card / Container    Card / Title    Card / Body    Card / Action Row
Hero / Background   Nav / Link Group   Button / Primary
```

---

## Rule 4 — Incremental building

One section per `use_figma` call. Validate via screenshot after each step. Return all created/mutated node IDs:

```js
return { created: { card: card.id, title: title.id } };
```
