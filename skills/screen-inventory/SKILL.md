---
name: screen-inventory
description: >
  Use this skill when the user wants to derive a list of screens, states, or UI moments from a set of use cases. Triggers include 'what screens do I need', 'extract the screens from these use cases', 'build a screen inventory', 'what UI states does this require', 'turn these use cases into a design backlog'. Also use automatically as part of the agentic use-case-to-design pipeline, between use-case-reviewer and excalidraw-generator or figma-spec. Do NOT use for writing use cases, generating designs, or producing code — this skill only extracts and classifies.
---

# Screen Inventory

Mechanically extracts every screen, state, and UI moment from a set of use cases and produces a structured inventory table.

## Overview

Every step in a use case's main success scenario that involves a user decision or a visible system response maps to a screen or a state. Every extension maps to at least one additional screen or state. This skill makes that extraction explicit before any design tool is opened.

The inventory is the bridge between specification and design. It tells the design team exactly what to build and keeps everything traceable back to the use case that required it.

## Extraction rules

Apply these rules to each use case systematically.

### Rule 1: Main success scenario steps

For each numbered step, ask: "Does this step involve the user seeing something or making a decision?"

- **Actor action steps** (user does something): map to the screen the user is looking at when they take the action
- **System response steps** (system does something visible): map to a screen state or transition
- **System internal steps** (system does something invisible): do not map to a screen — note as "no UI" and skip

Examples:
- "User opens the form URL" → Screen: Form view (initial/empty state)
- "System renders fields defined by the form builder" → State: Form view (fields loaded) — may collapse with above if instantaneous
- "User fills in fields and submits" → State: Form view (filled, submit action)
- "System writes a new record to the linked table" → No UI (internal)
- "System shows a confirmation message" → Screen: Confirmation screen

### Rule 2: Extensions

Every extension generates at least one additional screen or state. Apply these mappings:

- Validation failure extensions → error state on the current screen (not a new screen)
- Permission failure extensions → either an error state or a separate gate screen
- Alternate success path extensions → a variant confirmation screen or a status screen
- Empty state extensions (precondition not met) → a dedicated empty state screen
- Onboarding extensions (actor has no account, no workspace) → onboarding flow screen(s)

### Rule 3: Preconditions not met

For each precondition, generate an "empty state" entry: what the user sees when the precondition is not satisfied. These are often the most underdesigned screens in a product.

### Rule 4: Actor interfaces

Group screens by actor. If two actors see the same screen but with different data or controls, create two separate entries with a shared component reference.

## Output format

Produce a markdown table with these columns:

| Screen ID | Use case | Step / Extension | Actor | Screen name | Screen type | Notes |
|---|---|---|---|---|---|---|

**Screen ID:** S-[UC number]-[sequence]. E.g. S-02-01, S-02-02, S-02-03a.

**Use case:** UC-XX reference.

**Step / Extension:** The step number (e.g. "Step 2") or extension (e.g. "Ext 4a") that requires this screen.

**Actor:** The actor who sees this screen.

**Screen name:** A short descriptive name. Use the same vocabulary as the use case.

**Screen type:** One of: `full screen`, `state` (a variant of an existing screen), `modal`, `empty state`, `error state`, `confirmation`, `gate` (permission or onboarding barrier), `notification` (email, push, in-app).

**Notes:** Anything a designer needs to know — e.g. "collapses with S-02-01 if load is fast", "shares layout with S-03-02 but different data".

## After the table: derived outputs

After the inventory table, output three additional sections:

### Shared screens
List any screen IDs that serve multiple use cases. These are high-priority components — changes to them affect multiple flows.

### Empty states required
List every empty state entry and the precondition that triggers it. These are frequently missed in design and should be called out explicitly.

### Actor interface summary
For each actor, list how many screens they interact with and which use cases those screens belong to. This gives the design team a quick read of UI complexity per actor.

## Example output (partial)

| Screen ID | Use case | Step / Extension | Actor | Screen name | Screen type | Notes |
|---|---|---|---|---|---|---|
| S-02-01 | UC-02 | Step 1–2 | End user | Form view — empty | Full screen | Initial state before any fields are filled |
| S-02-02 | UC-02 | Step 3 | End user | Form view — filled | State | Variant of S-02-01 with values entered |
| S-02-03 | UC-02 | Step 6 | End user | Submission confirmation | Confirmation | Shown after successful write |
| S-02-04 | UC-02 | Ext 4a | End user | Form view — validation error | Error state | Required field highlighted; submit blocked |
| S-02-05 | UC-02 | Ext 4b | End user | Form view — type error | Error state | Type mismatch highlighted inline |
| S-02-06 | UC-02 | Ext 6a | End user | Submission confirmation — pending | Confirmation | Approval mode; different copy from S-02-03 |
| S-02-07 | UC-02 | Ext 6a | Workspace owner | Approval notification | Notification | Email or in-app; links to pending record |
