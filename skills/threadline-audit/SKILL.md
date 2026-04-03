---
name: threadline-audit
description: >
  Use this skill when the user wants to verify that the traceability chain between use cases, screens, tests, technical design, and code is intact. Triggers include 'audit the threadline', 'check for drift', 'is anything out of sync', 'what broke since the last change', 'run a drift check', 'what does this change affect', or any request to verify consistency across artefacts. Also use before releases or after modifying a use case. Do NOT use for fixing drift — this skill only detects and reports it for human decision.
---

# Threadline Audit

Detects drift between use cases and their downstream artefacts — screens, tests, designs, and code — so the team can decide what to update before the traceability chain silently degrades.

## Overview

Drift is inevitable. A developer refactors an interactor and drops the step-number comment. A test is patched to fix a flaky assertion, and the new assertion no longer matches the use case extension it claims to cover. A screen is added to the inventory during a design review but the use case is never updated to include the step that requires it.

None of these are bugs in isolation. The code works, the test passes, the screen looks correct. But the traceability chain — the core value of the Threadline — has broken. Over weeks, the specification and the implementation drift apart, and no single artefact can be trusted as the source of truth.

This skill reads the use cases and compares them against every downstream artefact. It does not fix anything. It surfaces drift so a human can decide whether to update the use case, regenerate the artefact, or accept the deviation.

## Modes

### Full audit

Check every use case against every artefact. Use this before releases or periodically (e.g. weekly) to catch accumulated drift.

### Change impact

Given a specific change to a use case (e.g. "Extension 6a of UC-02 was modified"), identify every artefact that depends on the changed element and classify each as REVIEW NEEDED or REGENERATE. Use this immediately after modifying a use case.

## Audit dimensions

### 1. Screen inventory against use cases

For each screen in the inventory, verify that its Step/Extension column references a step or extension that exists in the corresponding use case. For each step or extension in a use case that involves a visible user action or system response, verify that a screen entry exists.

Check for:
- Screens with no corresponding step or extension (orphaned screen)
- Steps or extensions with visible outcomes that have no screen entry (missing screen)
- Screen names that contradict the vocabulary used in the use case step

### 2. Test suite against use cases

For each acceptance test, verify that its name references a real use case ID, step, or extension. For each extension in a use case, verify that a corresponding test scenario exists.

Check for:
- Tests referencing extensions that do not exist (renamed or removed)
- Test assertions that contradict the expected outcome stated in the use case
- Extensions with no test coverage
- Happy path tests that skip steps present in the main success scenario

### 3. Technical design against use cases

For each interactor, entity, or value object in the technical design, verify that step-number comments trace back to real steps. Verify that invariant guards in domain entities match the preconditions and validation rules stated in the use case.

Check for:
- Interactor methods with no step-number comments
- Step-number comments referencing steps that no longer exist
- Invariant guards that do not match any stated precondition or extension
- Extension branches in the use case with no corresponding branch in the interactor

### 4. Figma spec against screen inventory

For each screen in the inventory, verify that a corresponding page or frame exists in the Figma spec. For each Figma frame, verify it maps to a screen ID.

Check for:
- Screen IDs missing from the Figma spec
- Figma frames with no corresponding screen ID (orphaned design)
- Screen type mismatches (inventory says "error state", Figma shows a full screen)

### 5. Code comments against step numbers

Scan implementation files for step-number comments (e.g. `// Step 3`, `// Ext 4a`). Verify each references a real step or extension in the corresponding use case.

Check for:
- Step-number comments referencing steps that were renumbered or removed
- Implementation branches for extensions with no step-number comment
- Files that should trace to a use case but contain no traceability comments

## Output: audit report

```
Threadline Audit Report

UC-[XX]: [Title]

  Screen inventory:
    ✓ [Screen ID] traces to [Step/Extension]
    ✗ [Screen ID] exists in inventory but has no corresponding step or extension
      → Possible drift: [explanation]

  Test suite:
    ✓ [Test name] covers [step/extension]
    ✗ [Test name]: [problem description]
      → Possible drift: [explanation]

  Technical design:
    ✓ [Method/guard] matches [precondition/step]
    ✗ [Method/location]: [problem description]
      → Possible drift: [explanation]
```

Report every check. Use ✓ for passing checks and ✗ for drift. Every ✗ line must include a "Possible drift" explanation stating what likely happened and which artefact is probably stale.

## Output: change impact report

When running in change impact mode, output:

```
UC-[XX] change detected: [description of change]

Affected artefacts:
  - [file]: [specific element] — REVIEW NEEDED
  - [file]: [specific element] — REGENERATE
```

Mark an artefact REVIEW NEEDED when the change may or may not invalidate it (e.g. a screen name might still be correct). Mark it REGENERATE when the change certainly invalidates it (e.g. a test that asserts the old extension outcome).

## When to run

- **Before releases:** full audit to catch accumulated drift
- **After use case changes:** change impact mode on the modified use case
- **Periodically:** weekly or biweekly full audit on active use cases
- **Before Gate 2:** verify that implementation artefacts still trace to the reviewed specification

## Common drift patterns

| Pattern | Typical cause | What drifts |
|---|---|---|
| Orphaned screen | Screen added during design review, use case not updated | Screen inventory has entries with no use case anchor |
| Stale test assertion | Test patched to fix flaky run, assertion no longer matches extension | Test passes but checks wrong outcome |
| Missing step comment | Interactor refactored, comments dropped or not updated | Code has no traceability link to use case |
| Renamed extension | Use case extension renumbered after review, downstream not updated | Tests and screens reference old extension IDs |
| Silent precondition change | Precondition relaxed in use case, invariant guard not updated | Domain entity rejects inputs the use case now allows |
| Figma orphan | Designer added exploratory frames, never mapped to inventory | Figma spec has frames that are not part of any flow |
