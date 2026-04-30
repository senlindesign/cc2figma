---
name: figma-qa-verifier
description: "Triggers after any use_figma call that creates or modifies nodes, or when user says 'verify', 'check bindings', 'QA'. Checks that every created node has proper Style/Variable bindings — catches silent fallbacks to raw values."
disable-model-invocation: false
---

# Figma QA Verifier Skill

This skill runs a lightweight, targeted post-creation audit. It does not deep-traverse the full node tree — it spot-checks only the specific node IDs explicitly returned from the preceding `use_figma` creation call. Its job is to catch one specific class of failure: Claude silently falling back to raw hardcoded values when style or variable binding fails.

**Prerequisite:** The preceding `use_figma` call must have returned `createdNodeIds` and/or `mutatedNodeIds`. If no node IDs are available, ask the user to re-run the creation step with a script that returns node IDs.

---

## When This Skill Triggers

- **Automatically** after any `use_figma` call that creates design nodes (the figma-use skill requires creation scripts to return node IDs — those IDs are the input to this skill)
- **User phrases:** "verify", "check bindings", "QA this", "did it bind correctly", "are the styles applied?", "check the variables"
- **After `figma-style-binding`** is applied — always run QA to confirm the bindings took effect

---

## Step 1 — Collect the Node IDs

From the previous `use_figma` result, extract the node IDs to audit:

```
createdNodeIds: ["123:456", "123:457", "123:458"]
mutatedNodeIds: ["100:12"]
```

Combine both lists into a single audit list. If neither list is present, surface this:

```
⚠️ No node IDs available from the preceding creation call.
   Ask Claude to re-run the creation script with return { createdNodeIds: [...] }.
```

---

## Step 2 — Run the Verification Script

Load the `figma-use` skill, then call `use_figma` with the following verification script. Pass the collected node IDs as an array literal in the script.

```javascript
// Paste the actual node ID array here before running
const nodeIdsToAudit = ["123:456", "123:457", "123:458"];

const results = [];

for (const id of nodeIdsToAudit) {
  const node = await figma.getNodeByIdAsync(id);

  if (!node) {
    results.push({ id, status: "ERROR", reason: "Node not found — may have been deleted" });
    continue;
  }

  const checks = [];

  // --- TEXT STYLE CHECK ---
  // Applies to: TextNode
  if (node.type === "TEXT") {
    const bound = node.textStyleId && node.textStyleId !== "";
    checks.push({
      property: "textStyleId",
      bound,
      value: bound ? node.textStyleId : "(none)",
      rule: "Text nodes must have textStyleId set to a Style ID"
    });
  }

  // --- FILL STYLE / VARIABLE CHECK ---
  // Applies to: any node that has fills (TEXT, FRAME, RECTANGLE, ELLIPSE, COMPONENT, INSTANCE, etc.)
  if ("fills" in node && Array.isArray(node.fills) && node.fills.length > 0) {
    const hasFillStyleId = "fillStyleId" in node && node.fillStyleId && node.fillStyleId !== "";
    const hasBoundFillVar =
      node.boundVariables &&
      node.boundVariables.fills &&
      node.boundVariables.fills.length > 0;

    const bound = hasFillStyleId || hasBoundFillVar;
    checks.push({
      property: "fills",
      bound,
      value: hasFillStyleId
        ? `fillStyleId: ${node.fillStyleId}`
        : hasBoundFillVar
        ? `boundVariables.fills: [${node.boundVariables.fills.map(f => f?.id ?? "?").join(", ")}]`
        : `RAW fill — ${JSON.stringify(node.fills[0]?.color ?? node.fills[0])}`,
      rule: "Fills must use fillStyleId (Paint Style) or boundVariables.fills (COLOR Variable)"
    });
  }

  // --- AUTO LAYOUT SPACING CHECK ---
  // Applies to: FrameNode / ComponentNode with layoutMode !== NONE
  if ("layoutMode" in node && node.layoutMode !== "NONE") {
    const spacingProps = ["paddingLeft", "paddingRight", "paddingTop", "paddingBottom", "itemSpacing"];
    for (const prop of spacingProps) {
      if (prop in node) {
        const bound = node.boundVariables && prop in node.boundVariables;
        checks.push({
          property: prop,
          bound,
          value: bound
            ? `boundVariables.${prop}: ${node.boundVariables[prop]?.id ?? "?"}`
            : `RAW ${node[prop]}px`,
          rule: `Auto Layout spacing must use setBoundVariable("${prop}", variable)`
        });
      }
    }
  }

  // --- CORNER RADIUS CHECK ---
  // Applies to: any node with cornerRadius property that is non-zero
  if ("cornerRadius" in node && typeof node.cornerRadius === "number" && node.cornerRadius > 0) {
    const bound = node.boundVariables && "cornerRadius" in node.boundVariables;
    checks.push({
      property: "cornerRadius",
      bound,
      value: bound
        ? `boundVariables.cornerRadius: ${node.boundVariables.cornerRadius?.id ?? "?"}`
        : `RAW ${node.cornerRadius}px`,
      rule: 'cornerRadius must use setBoundVariable("cornerRadius", variable)'
    });
  }

  const failed = checks.filter(c => !c.bound);
  const passed = checks.filter(c => c.bound);

  results.push({
    id,
    name: node.name,
    type: node.type,
    status: failed.length === 0 ? "PASS" : "FAIL",
    passed: passed.map(c => c.property),
    failed: failed.map(c => ({
      property: c.property,
      found: c.value,
      rule: c.rule
    }))
  });
}

return { auditResults: results };
```

---

## Step 3 — Parse and Output the QA Report

After `use_figma` returns `auditResults`, format and display the report using this structure:

```
── Figma QA Report ────────────────────────────────────────
Audited: [N] nodes   Passed: [N]   Failed: [N]
───────────────────────────────────────────────────────────

✅ PASS  "Button/Primary" (FRAME) — 123:456
         textStyleId ✓  fillStyleId ✓  paddingLeft ✓  paddingRight ✓

❌ FAIL  "Label" (TEXT) — 123:457
         textStyleId ✗  →  No Text Style bound. Raw font properties detected.
                          Rule: Text nodes must have textStyleId set to a Style ID.

❌ FAIL  "Card" (FRAME) — 123:458
         paddingTop ✗  →  RAW 24px — no bound variable.
                          Rule: Auto Layout spacing must use setBoundVariable.
         cornerRadius ✗  →  RAW 8px — no bound variable.
                          Rule: cornerRadius must use setBoundVariable.

───────────────────────────────────────────────────────────
```

If all nodes pass:

```
── Figma QA Report ────────────────────────────────────────
✅ All [N] nodes passed binding verification.
   Every text style, fill, spacing, and corner radius is
   properly bound to Figma Styles or Variables.
───────────────────────────────────────────────────────────
```

---

## Step 4 — Fix Loop (for failed nodes)

For each FAIL, Claude must fix the binding before considering the task complete. Do not continue to any next design step until all QA failures are resolved.

### Fix protocol per failure type

**textStyleId missing:**
1. Consult the Style Registry from the current session (loaded by `figma-preflight`).
2. Find the best matching Text Style by semantic role (heading, body, label, etc.).
3. Run a targeted fix script:
   ```javascript
   const node = await figma.getNodeByIdAsync("123:457");
   await figma.loadFontAsync(node.fontName);
   node.textStyleId = "<correct style ID from Style Registry>";
   return { fixed: node.id, textStyleId: node.textStyleId };
   ```

**fillStyleId / boundVariables.fills missing:**
1. Consult the Variable Registry and Style Registry.
2. Determine whether a Paint Style or COLOR Variable is appropriate.
3. Run a targeted fix script:
   ```javascript
   const node = await figma.getNodeByIdAsync("123:457");
   // Option A — Paint Style
   node.fillStyleId = "<id from Style Registry>";
   // Option B — Variable (preferred)
   const variable = await figma.variables.getVariableByIdAsync("<id from Variable Registry>");
   const fill = { type: "SOLID", color: { r: 0, g: 0, b: 0 } };
   node.fills = [figma.variables.setBoundVariableForPaint(fill, "color", variable)];
   return { fixed: node.id };
   ```

**Auto Layout spacing (paddingLeft/Right/Top/Bottom, itemSpacing) unbound:**
1. Find the matching FLOAT Variable from the Variable Registry by semantic scope (`PADDING`, `GAP`).
2. Run a targeted fix script:
   ```javascript
   const node = await figma.getNodeByIdAsync("123:458");
   const variable = await figma.variables.getVariableByIdAsync("<id from Variable Registry>");
   node.setBoundVariable("paddingTop", variable);
   node.setBoundVariable("paddingBottom", variable);
   node.setBoundVariable("paddingLeft", variable);
   node.setBoundVariable("paddingRight", variable);
   return { fixed: node.id };
   ```

**cornerRadius unbound:**
1. Find the matching FLOAT Variable from the Variable Registry by scope (`CORNER_RADIUS`).
2. Run a targeted fix script:
   ```javascript
   const node = await figma.getNodeByIdAsync("123:458");
   const variable = await figma.variables.getVariableByIdAsync("<id from Variable Registry>");
   node.setBoundVariable("cornerRadius", variable);
   return { fixed: node.id };
   ```

### After each fix, re-run the audit

After applying all fixes, re-run the verification script from Step 2 on the previously failing node IDs only. Confirm they now show `PASS`. Output a concise re-check report:

```
── Re-check ───────────────────────────────────────────────
✅ "Label" (TEXT) — 123:457  textStyleId now bound ✓
✅ "Card" (FRAME) — 123:458  paddingTop ✓  cornerRadius ✓
All previously failing nodes now pass.
───────────────────────────────────────────────────────────
```

If a fix fails (binding still not applied after the fix script), escalate to the user:

```
⚠️ Fix attempt failed for "Label" (TEXT) — 123:457
   textStyleId still empty after applying Style ID [id].
   Possible causes: font not loaded, style ID invalid, node is inside a locked component.
   Action required: please check the node in Figma and confirm the style exists.
```

---

## What Claude Must Never Do in the Fix Loop

| Forbidden | Required instead |
|---|---|
| Skipping QA because "the script looked correct" | Always run the verification script after creation |
| Marking a node PASS when `textStyleId` is an empty string `""` | Treat empty string as unbound — same as null |
| Applying a raw hex color as a temporary fix | Only bind to a Style or Variable — never introduce new raw values during a fix |
| Moving on to the next design task while failures remain | All FAIL nodes must be resolved before proceeding |
| Re-running the full creation script to fix one property | Use targeted single-node fix scripts to avoid clobbering adjacent bindings |

---

## Plugin API Quick Reference

| What to check | Property / Method | Bound when... |
|---|---|---|
| Text Style | `node.textStyleId` | Non-empty string (`"S:abc123..."`) |
| Fill Style | `node.fillStyleId` | Non-empty string (`"S:abc123..."`) |
| Fill Variable | `node.boundVariables.fills` | Array with at least one entry |
| Padding (Auto Layout) | `node.boundVariables.paddingLeft` etc. | Key exists in `boundVariables` |
| Gap (Auto Layout) | `node.boundVariables.itemSpacing` | Key exists in `boundVariables` |
| Corner Radius | `node.boundVariables.cornerRadius` | Key exists in `boundVariables` |
| Stroke Style | `node.strokeStyleId` | Non-empty string |
| Stroke Variable | `node.boundVariables.strokes` | Array with at least one entry |

**Note:** `node.boundVariables` is an object. A key's presence (not just truthiness) indicates a bound variable. Always use `prop in node.boundVariables` to test — do not use `node.boundVariables[prop]` alone, as a key with a falsy value could give a false negative.

---

## Scope of This Skill

This skill performs **spot-checking**, not deep tree traversal. It audits exactly the nodes whose IDs were returned by the preceding `use_figma` creation call — not the entire document, not descendant trees. This keeps the audit fast and focused.

For a full document-wide audit, use `get_variable_defs` from the Figma MCP tools on a broader node selection. That is outside the scope of this skill.
