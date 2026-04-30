---
name: reference-interpreter
description: "Triggers when user shares a screenshot, image, URL, or design description — or says 'analyze this', 'make a brief', 'interpret this reference'. Outputs a structured Design Brief mapping visual intent to design system tokens. Waits for 'confirmed' before designing."
disable-model-invocation: false
---

# Skill: reference-interpreter

## Purpose

Before any design work begins, analyze a reference (screenshot, URL, or written description) and produce a structured **Design Brief**. The Brief maps visual observations directly to the design system tokens and styles available in the current session's Token Map and Style Registry (loaded by `figma-preflight`). Output the Brief, then stop and wait for the user to confirm before placing a single frame in Figma.

The output schema is structured so that `figma-generate-design` can consume the confirmed Brief directly as its input.

---

## Trigger Conditions

Load and execute this skill when any of the following occur:

- User shares a screenshot or image file
- User shares a URL (webpage, landing page, app screen)
- User provides a written design description
- User says any of: "analyze this reference", "interpret this", "make a brief", "what should we build", "analyze this design", "build something like this"
- At the start of a design session, after `figma-preflight` has passed, when a design direction is provided

---

## Execution Flow

### Phase 1 — Analyze the Reference

Examine the reference (image, URL content, or description) and document the following six dimensions. Be specific and visual — describe what you actually observe, not what you assume.

**1. Layout Rhythm**
- Overall structure: full-bleed vs contained layout
- Number of columns per section
- Section heights (viewport-height, fixed, content-driven)
- Grid width (e.g. 1360px max-width, edge-to-edge)

**2. Typography Strategy**
- Heading size relative to body copy (ratio, prominence)
- Weight contrast between hierarchy levels
- Tracking/letter-spacing usage (tight, normal, wide)
- Number of distinct hierarchy levels visible (h1, h2, body, label, caption, etc.)
- Any display or decorative type treatments

**3. Color Strategy**
- Dark vs light sections and their distribution
- Accent color: how frequently used, what it marks (CTA, highlight, icon)
- Neutral dominance: is the palette minimal or expressive
- Number of distinct background tones used

**4. Spacing Cadence**
- Overall feel: generous/airy vs compact/dense
- Section padding (top/bottom rhythm)
- Internal component gap (card padding, list spacing)
- Consistency: does spacing feel systematic or varied

**5. Visual Anchor**
- What draws the eye in each major section
- Is it: large type, hero image, illustration, icon cluster, data visualization, graphic element
- Position of anchor: centered, left, right, offset

**6. Alignment Pattern**
- Text alignment per section: left, center, right
- Is layout symmetric or asymmetric
- CTA/button alignment relative to headline

---

### Phase 2 — Map to Design System

For each observation from Phase 1, identify the specific Token or Style from the session's **Token Map** and **Style Registry** (established by `figma-preflight`) that will deliver it.

Format each mapping as:

```
Observation → Token/Style mapping
"Large dark headline" → Text Style: heading/h1 · Color Variable: text/primary
"Neutral section background" → Variable: background/surface-2
"Tight card spacing" → Gap Variable: gap/xs · Padding Variable: padding/sm
"Accent CTA button" → Color Variable: accent/primary · Text Style: label/button
```

If no token or style exists for an observation, flag it explicitly as a gap:

```
⚠️ Gap: reference uses a gradient background — no gradient token in Variable Registry.
   Options: (a) use nearest solid color token: background/surface-3
             (b) add a gradient token before proceeding
   Awaiting user decision.

⚠️ Gap: reference uses a custom display typeface — not present in Style Registry.
   Options: (a) substitute nearest available heading style: heading/h1
             (b) add the typeface to the design system first
   Awaiting user decision.
```

Every observation must either map to a token/style or be flagged. Do not silently skip unmatched observations or fall back to hardcoded values.

---

### Phase 3 — Output the Design Brief

Output the full Design Brief using the exact schema below. This schema is structured for direct consumption by `figma-generate-design`.

```
## Design Brief

**Reference**: [source — file name, URL, or "written description"]
**Section**: [what section or screen this covers, e.g. "Hero section", "Features grid", "Full landing page"]
**Aesthetic keywords**: [3–5 words that capture the visual mood, e.g. "minimal, editorial, high-contrast"]

---

### Layout
- Structure: [e.g. "2-column asymmetric, 1360px max-width grid"]
- Section height: [e.g. "full-viewport hero, contained features section, full-bleed footer"]
- Alignment: [e.g. "left-aligned text, centered CTA, right-offset image"]

### Typography
- Heading: [Text Style name from Style Registry] — [reason this style fits the observation]
- Body: [Text Style name from Style Registry] — [reason]
- Label/caption: [Text Style name if applicable, or "none required"]

### Colors
- Background: [Variable name from Variable Registry] — [which section/context]
- Primary text: [Variable name]
- Secondary text: [Variable name, if applicable]
- Accent: [Variable name] — [used for: CTA / highlight / icon / border]
- Surface/card: [Variable name, if applicable]

### Spacing
- Section padding (vertical): [Variable name]
- Section padding (horizontal): [Variable name]
- Internal gap (between elements): [Variable name]
- Card/component internal padding: [Variable name, if applicable]

### Components needed
- [Component name]: [from library? yes/no — library name if yes]
- [Component name]: [from library? yes/no]
- [Component name]: [from library? yes/no]

### Tokens not covered (gaps)
- [Gap description] — awaiting user decision: [option a] or [option b]
- (none — all observations mapped) [use this line if no gaps exist]

---

### Notes for figma-generate-design
[Any layout instructions, ordering of sections, special behaviors, or constraints that the generating skill should know before building. E.g. "Build hero first, then features grid. Hero uses full-bleed dark background with left-aligned text. Features grid uses 3-column card layout with surface-1 background."]
```

---

### Phase 4 — Wait for Confirmation

After outputting the complete Design Brief, output exactly this line and nothing else:

```
Brief complete. Review the mapping above — type `confirmed` to begin building, or tell me what to adjust.
```

Then stop. Do NOT:
- Place any frames or nodes in Figma
- Call `use_figma` or any design tool
- Proceed with layout, components, or variables
- Make any assumptions about approval

**If the user modifies the Brief**, apply the changes, output the updated Brief in full, and wait again with the same confirmation prompt.

**Only when the user types exactly `confirmed`** should you proceed to invoke `figma-generate-design` (or begin the building phase), passing the confirmed Brief as structured input.

---

## Handoff to figma-generate-design

When the user confirms, the Design Brief becomes the structured input for `figma-generate-design`. The following fields are consumed directly:

| Brief field | Consumed as |
|---|---|
| Layout → Structure | Grid setup, frame width, column count |
| Layout → Section height | Frame height constraints |
| Typography → Heading/Body/Label | Text Style bindings via `figma-style-binding` |
| Colors → Background/Text/Accent | Variable bindings via `figma-style-binding` |
| Spacing → Section padding / Internal gap | Auto-layout gap and padding variable bindings |
| Components needed | Library import list for `importComponentByKeyAsync` |
| Tokens not covered (gaps) | Flagged before build begins, resolved by user |
| Notes for figma-generate-design | Section build order and special instructions |

---

## Error Handling

**If `figma-preflight` has not been run:**
Output a warning before proceeding with Phase 1:
```
⚠️ figma-preflight has not been run in this session. Token Map and Style Registry are unavailable.
   Phase 2 mappings will use placeholder names rather than confirmed token IDs.
   Run preflight on your Figma file before confirming the Brief to ensure accurate bindings.
```
Then continue with Phase 1–3 using descriptive placeholder names, and flag all mappings as unverified.

**If the reference is ambiguous or incomplete:**
Ask one focused clarifying question before proceeding. Do not ask multiple questions at once. Example:
```
Before I analyze: is this a full landing page, or a specific section (e.g. hero, features, pricing)?
```

**If no design system tokens are available:**
Flag every mapping as a gap and surface the decision to the user before confirming.
