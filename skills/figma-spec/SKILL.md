---
name: figma-spec
description: >
  Use this skill when the user wants a Figma file specification derived from a screen inventory and component inventory. Triggers include 'write a Figma spec', 'how should I structure the Figma file', 'create the Figma handoff document', 'what pages should the Figma file have', 'produce the design spec'. Also use automatically in the agentic pipeline after component-inventory. Output is a structured markdown document a designer can execute directly. Do NOT use before screen-inventory and component-inventory exist — this skill depends on both.
---

# Figma Spec

Produces a structured Figma file specification from a screen inventory and component inventory. The output is a markdown document a designer executes in Figma — not a Figma file itself.

## Overview

The Figma spec answers: "How should this design file be organised so that it stays traceable to the use cases?" It defines page structure, screen layout, component organisation, prototype flow wiring, and annotation conventions.

The guiding rule: **the spatial layout of the Figma file is the documentation.** Anyone opening the file should be able to read the design the same way they read the use cases — left to right, happy path on top, extensions below.

## Output structure

The spec has six sections. Produce all six.

---

### Section 1: File structure

List the pages the Figma file should contain, in order.

**Page order:**

1. `_Cover` — project name, use case suite name, last updated date, owner
2. `_Components` — the component library (all components from component-inventory)
3. One page per use case, named by ID and title: `UC-01 · Create shared table`
4. `_Archive` — deprecated screens moved here rather than deleted

**Rules:**
- Pages prefixed with `_` sort to the top in Figma's panel
- Use case pages are ordered by dependency: use cases that are preconditions for others come first
- Never put screens from multiple use cases on the same page — traceability breaks

---

### Section 2: Component page layout

Specify the layout of the `_Components` page.

**Structure (top to bottom, left to right):**

For each component from the component inventory:
1. Component heading text (component ID + name, e.g. "C-01 · FormFieldBlock")
2. A row of all variants, spaced 40px apart, left to right in this order:
   - Default/empty state first
   - Filled/active states
   - Error states (left to right: least severe to most severe)
   - Disabled/hidden/permission states
   - Actor-specific variants (group by actor, label above each group)
3. A variant label below each instance (the variant name, e.g. `required-error`)
4. A use case reference below the label (e.g. `UC-02 ext 4a`)

**Component groups:** separate component groups with a 120px vertical gap. Add a section label above each group using Figma's section frame feature.

---

### Section 3: Use case page layout

Specify the layout of each use case page. Apply this layout consistently to all use case pages.

**Canvas layout:**

```
[Title frame — use case ID + title + actor list]

[Main success scenario — screens left to right, 80px gap between screens]

    S-XX-01  →  S-XX-02  →  S-XX-03  →  ...

[Extension screens — below the happy path, 60px below, branching from their parent step]

              ↓ (ext 4a)        ↓ (ext 4b)
           S-XX-04           S-XX-05
```

**Layout rules:**
- Screens are arranged left to right in scenario step order — same sequence as the numbered use case steps
- Extension screens are placed directly below the step they branch from, offset 60px below the bottom of the happy path row
- Use Figma's section frames to group: one section labelled "Happy path", one labelled "Extensions"
- Each screen frame is named by Screen ID: `S-02-01`, `S-02-02`, etc.
- Actor changes are indicated by a coloured label strip at the top of each frame: blue for primary actor, gray for system, green for secondary actor

**Flow arrows (connectors):**
- Figma connectors join screens in the prototype tab
- Happy path: one flow named `[UC-XX] Happy path`, tracing all main success scenario steps
- Each extension: one flow named `[UC-XX] Ext Xa`, branching from the parent step's screen
- Label connectors with the step number or extension ID

---

### Section 4: Annotation conventions

Specify the annotation layer for each screen frame.

Every screen frame has an annotation overlay — a separate locked frame positioned above the screen, containing:

```
[Screen ID]  ·  [Use case ID]  ·  [Step or Extension]  ·  [Screen type]
```

Example:
```
S-02-04  ·  UC-02  ·  Ext 4a  ·  Error state
```

**Annotation placement:** 12px above the top of the screen frame, left-aligned.
**Annotation style:** 11px, color `#888888`, not bold.
**Annotation layer:** locked, named `_annotations`. Never merged with screen content.

This ensures the annotation survives screen redesigns and is not accidentally moved or deleted.

---

### Section 5: Handoff checklist

Include this checklist in the Figma spec. The designer checks every item before marking the file ready for handoff.

**Component coverage:**
- [ ] Every component in the component inventory exists on the `_Components` page
- [ ] Every variant in the component inventory exists as a Figma variant
- [ ] Variant names match the component inventory exactly (condition-based, not treatment-based)
- [ ] Actor-specific variants are labelled with the actor name

**Screen coverage:**
- [ ] Every screen in the screen inventory has a frame on the correct use case page
- [ ] Every screen frame is named by its Screen ID
- [ ] Every screen frame has an annotation

**Prototype flows:**
- [ ] Every use case page has a `Happy path` flow
- [ ] Every extension has its own named flow
- [ ] Flows are labelled with use case ID and step/extension reference

**Traceability:**
- [ ] Every screen can be traced back to a use case step or extension via its annotation
- [ ] Every component variant can be traced back to a use case condition via the reference on the component page

**Final check:**
- [ ] Return to the use case documents — verify every step and every extension has a screen or state in the Figma file
- [ ] Any step or extension without a visual answer is flagged as a gap, not silently ignored

---

### Section 6: Handoff package

Specify what is delivered to developers at handoff.

The handoff package contains three artifacts that all point at the same truth:

1. **Use case documents** — the spec (source of truth)
2. **Excalidraw flow diagrams** — the navigation model (one per use case)
3. **Figma file** — the visual implementation

Developers receive all three. When reviewing the Figma file, they should be able to open the corresponding use case document and Excalidraw diagram side by side and verify that every step has a visual answer.

Instructions for developers:
- Screen IDs in Figma annotations match Screen IDs in the screen inventory
- Screen IDs in the screen inventory reference use case steps and extensions
- Use case steps and extensions are in the use case documents
- Excalidraw diagrams show the navigation model in the same step order as the use case

The chain is: **Figma frame → Screen ID → Use case step → Use case document.** Every design decision is traceable to a requirement.
