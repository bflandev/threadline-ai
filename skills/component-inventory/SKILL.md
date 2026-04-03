---
name: component-inventory
description: >
  Use this skill when the user wants to derive a UI component library from a screen inventory or a set of use cases. Triggers include 'what components do I need', 'derive the component library', 'extract shared components', 'build a component inventory', 'what Figma components should I create'. Also use automatically in the agentic pipeline after screen-inventory and before figma-spec. Do NOT use before a screen inventory exists — run screen-inventory first.
---

# Component Inventory

Derives a structured component library from a screen inventory, identifying shared UI elements and their variants.

## Overview

A component inventory answers: "What are the reusable pieces that compose all the screens, and what states does each piece need to support?" It sits between the screen inventory (what screens exist) and the Figma spec (how to organise those screens in a design tool).

The key principle: **name variants after conditions, not visual treatments.** A component variant named `required-empty` is traceable to a use case extension. A variant named `red-outline` is not.

## Step 1: Extract raw components from screens

For each screen in the inventory, list every distinct UI element visible on that screen. Do not design yet — just name what is there.

Common elements to look for:
- Input fields (text, number, date, select, textarea, checkbox)
- Buttons (primary action, secondary action, destructive)
- Record rows (in a table or list view)
- Status badges or tags
- Navigation items (sidebar, tabs, breadcrumb)
- Empty state containers
- Confirmation or notification banners
- Modal or overlay containers
- Permission gates (login prompt, access denied message)
- Data view blocks (table, gallery, calendar)
- Form field blocks (label + input + error message)

## Step 2: Consolidate into named components

Group elements that represent the same logical thing across different screens. A form field in UC-02 and a form field in UC-05 are the same component even if they have different labels.

Name components as noun phrases describing what they are, not where they appear:
- `FormFieldBlock` not `EventSubmissionField`
- `RecordRow` not `EventTableRow`
- `StatusBadge` not `ConfirmedGreenPill`

## Step 3: Define variants for each component

For each component, list every state it can be in based on the screen inventory. Derive states from:

- **Main success scenario states:** the component as it appears in the happy path (e.g. `FormFieldBlock/empty`, `FormFieldBlock/filled`)
- **Extension states:** the component as it appears in each extension (e.g. `FormFieldBlock/required-error`, `FormFieldBlock/type-error`)
- **Permission states:** what the component looks like for a restricted actor (e.g. `TableColumn/hidden`, `TableColumn/view-only`)
- **Empty states:** what the component looks like before any data exists (e.g. `DataViewBlock/no-table-linked`, `RecordRow/skeleton`)

## Output format

Produce a component table, then a variants table for each component.

### Component table

| Component ID | Component name | Description | Used in screens | Use case refs |
|---|---|---|---|---|
| C-01 | FormFieldBlock | A labelled input with optional error message | S-02-01, S-02-02, S-02-04, S-02-05 | UC-02 steps 2–4 |
| C-02 | StatusBadge | Coloured pill indicating record status | S-01-01, S-03-02 | UC-01, UC-03 |

### Variants table (one per component)

**C-01 FormFieldBlock**

| Variant name | Trigger | Visual note | Use case ref |
|---|---|---|---|
| `empty` | Field rendered, no input yet | Label + placeholder | UC-02 step 2 |
| `filled` | User has entered a value | Label + value | UC-02 step 3 |
| `required-error` | Submit attempted, required field empty | Label + red highlight + error message | UC-02 ext 4a |
| `type-error` | Value fails type validation | Label + red highlight + type error message | UC-02 ext 4b |
| `disabled` | Field is visible but not editable | Label + grayed value, no focus | UC-04 ext 3a |
| `hidden` | Field is restricted from this actor | Not rendered | UC-04 step 5 |

## Step 4: Identify shared components

After producing all variant tables, list components that appear in three or more screens. These are the highest-priority components to design first — changes to them propagate widely.

Flag: "Shared components — design these before composing any screens:"

## Step 5: Identify actor-specific variants

Some components need a variant per actor, not per state. Flag these explicitly:

"The following components require actor-specific variants — the same data is presented differently depending on who is viewing it:"

Example:
- `TableColumn` — owner sees all columns; collaborator sees restricted columns hidden; public sees only published fields

This prevents the design team from accidentally designing one version and assuming it works for all actors.

## Naming rules summary

| Do | Don't |
|---|---|
| `FormFieldBlock/required-error` | `FormFieldBlock/red` |
| `RecordRow/restricted` | `RecordRow/grayed-out` |
| `StatusBadge/pending` | `StatusBadge/amber-pill` |
| `TableColumn/hidden` | `TableColumn/invisible` |
| `ConfirmationScreen/pending` | `ConfirmationScreen/variant-2` |

Condition-based names survive a visual redesign. Treatment-based names do not.
