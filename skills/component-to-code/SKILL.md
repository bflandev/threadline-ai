---
name: component-to-code
description: >
  Use this skill when the user wants to translate a component inventory into framework-specific component specifications. Triggers include 'generate component specs', 'turn components into code', 'derive props from the inventory', 'spec the frontend components'. Also use when constraining AI-assisted frontend generation. This skill is optional in the pipeline — use after component-inventory and screen-inventory. Do NOT use before a component inventory exists.
---

# Component to Code

Translates a component inventory into framework-specific component specifications — props, state mappings, and screen ID traceability — for implementation or AI-assisted code generation.

## Overview

The component inventory defines *what* exists and *what states* it supports. This skill bridges to *how those components are expressed in code*. Without it, traceability from use case to UI breaks at the code boundary.

Key principle: **every variant maps to a concrete prop combination, and every prop combination traces back to a screen ID and use case step.**

## Dependencies

- **Component inventory** — component and variant tables (C-XX identifiers, variant names, use case refs)
- **Screen inventory** — screen table (S-XX-YY identifiers) for traceability

## Step 1: Derive props from variant dimensions

Extract the dimensions that change between variants. Each dimension becomes a prop.

| Variant pair | What changes | Prop | Type |
|---|---|---|---|
| `empty` vs `filled` | presence of value | `value` | `string \| null` |
| `required-error` vs `type-error` | error kind | `error` | `{ code: string, message: string } \| null` |
| `disabled` vs active | interactivity | `disabled` | `boolean` |

Static props needed for rendering (e.g. `fieldId`, `label`, `type`) come from the component description and screen context.

## Step 2: Map variants to state combinations

Each variant becomes a named state expressed as a specific prop combination — the contract between design and code.

## Step 3: Attach screen ID traceability

Every state must reference the screen ID it was derived from, linking back to the use case step or extension via the screen inventory.

## Component Spec Template

One spec block per component:

```
Component: FormFieldBlock
Framework: React + TypeScript

Props:
  fieldId: string
  label: string
  type: "text" | "number" | "date" | "select"
  required: boolean
  value: string | null
  error: { code: string, message: string } | null
  disabled: boolean

States (derived from component inventory variants):
  FormFieldBlock/empty          -> value=null, error=null
  FormFieldBlock/filled         -> value=string, error=null
  FormFieldBlock/required-error -> value=null, error={ code: "required", ... }
  FormFieldBlock/type-error     -> value=string, error={ code: "type_mismatch", ... }
  FormFieldBlock/disabled       -> disabled=true

Derived from:
  S-02-01 (empty), S-02-02 (filled), S-02-04 (ext 4a), S-02-05 (ext 4b)
```

## Framework-Specific Guidance

### React + TypeScript (primary)

- Props as a TypeScript `interface` or `type`. Use discriminated unions for mutually exclusive states.
- `null` over `undefined` for absent values — explicit and serialisable.
- Each named state should have a Storybook story rendering that exact prop combination.

### Vue 3 + TypeScript

- Props via `defineProps<T>()` with the same interface. Named states map to Histoire/Storybook entries.

### Svelte

- Props as exported `let` declarations with TypeScript. Variant states map to reactive `$:` statements.

The spec is framework-agnostic — only prop declaration syntax differs.

## Constraining AI-Assisted Frontend Generation

This spec serves the same role for frontend that the TDD test suite serves for backend: **it defines the target before generation begins.**

Provide the spec as context to the AI tool. Generated code must:

1. Accept exactly the declared props (no extra, no missing)
2. Handle every named state (each variant reachable via props)
3. Reference screen IDs in code comments for traceability

A component that adds undeclared props or ignores a variant has drifted from the spec. Reject it like code that fails a test.

## Output Format

Per component: (1) the spec block, (2) a state table mapping variants to props, (3) screen ID cross-references.

### Verification Checklist

- [ ] Every component in the inventory has a spec block
- [ ] Every variant in the inventory maps to a named state with explicit prop values
- [ ] Every state references at least one screen ID
- [ ] Every screen ID traces back to a use case step or extension
- [ ] Props use `null` for absent values, not `undefined`
- [ ] No variant is left unmapped (compare variant count to state count)
- [ ] Shared components (used in 3+ screens) are flagged for priority review

## Common Mistakes

- **Inventing props not grounded in variants** — every prop must trace to a dimension that changes between at least two variants. No variant, no prop.
- **Losing traceability at the code boundary** — screen ID and use case references must survive into the spec. Without them, the component is a guess.
- **Specifying visual treatments instead of states** — `error={ code: "required" }` is testable; `borderColor="red"` is not. Keep it behavioural.
- **Assuming one framework fits all** — adapt prop declaration syntax per framework but do not change the state mapping.
- **Skipping disabled/hidden variants** — permission-driven states (disabled, hidden, view-only) must appear. They are easy to forget and expensive to retrofit.
