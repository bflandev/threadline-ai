---
name: use-case-reviewer
description: >
  Use this skill when the user wants to review, audit, or improve an existing use case or set of use cases before using them for development or design. Triggers include 'review this use case', 'check this spec', 'what is missing', 'is this good enough to build from', 'find the gaps', or any request to validate a use case document. Also use proactively before passing use cases to code generation — run this skill first and surface issues before a line of code is written. Do NOT use for writing new use cases from scratch — that is the use-case-writer skill.
---

# Use Case Reviewer

Audits existing use cases and surfaces ambiguities, gaps, and structural problems before they become code bugs or design gaps.

## Overview

The purpose of this skill is to answer the question an AI code generator would ask if it could: "What in this use case would prevent me from implementing it correctly?" Run this skill between writing use cases and generating any downstream artifact — code, tests, screens, or diagrams.

The output is a structured review with specific, actionable issues. It is not a rewrite. Issues should be flagged so the author can resolve them, not silently fixed.

## Review dimensions

Evaluate each use case across six dimensions. For each issue found, output: the dimension, the location (e.g. "Step 3", "Extension 4a", "Preconditions"), the problem, and a suggested resolution.

---

### Dimension 1: Implementability

Ask: "Could a developer implement this step without making any assumptions not stated in the use case?"

Flags:
- Steps that describe the mechanism, not the outcome ("System calls the API" — remove the mechanism)
- Steps that are ambiguous about who acts ("The form is submitted" — who submits it?)
- Steps that mix multiple actions ("User fills in the form and clicks submit and sees confirmation" — split these)
- Outcomes that are not verifiable ("System processes the request" — processes it how? To what end?)

---

### Dimension 2: Extension completeness

Ask: "For every step in the main success scenario, what can go wrong, and has it been captured as an extension?"

Systematically ask these questions at each step:
- Can the actor's action fail? (network error, timeout, permission denied)
- Can the system's response be something other than success? (not found, conflict, invalid state)
- Is there a validation rule that is not modelled? (required field, type constraint, uniqueness)
- Is there a permission boundary? (user lacks the role to perform this action)
- Is there an alternate success path? (the goal is achieved but via a different route)

Flag any step that has no extensions and no plausible explanation for why failures are impossible.

---

### Dimension 3: Precondition completeness

Ask: "What must be true for step 1 to be reachable, and is all of it stated?"

Flags:
- Preconditions that are implied but not stated ("User is on the form page" — how did they get there? Is there a prior use case?)
- Preconditions whose absence is not handled ("At least one table exists" — what happens in the empty state?)
- Preconditions that are vague ("User has access" — access to what, at what permission level?)

For each precondition whose absence is not covered by an extension, flag: "Empty state required — no design or extension handles the case where [precondition] is not met."

---

### Dimension 4: Postcondition observability

Ask: "Could a tester verify each postcondition by observing the system, without reading the source code?"

Flags:
- Postconditions describing internal state ("Record is written to the database" → "A new record appears in the table view")
- Postconditions that are missing for affected parties ("Submitter receives confirmation" is stated, but "Owner is notified" is not — is that intentional?)
- Postconditions that contradict an extension ("Submitter receives confirmation" but extension 6a routes to a pending state — does the submitter receive confirmation in that case too, or a different message?)

---

### Dimension 5: Actor consistency

Ask: "Is every actor who appears in the steps identified in the actor list, and is every listed actor used?"

Flags:
- An actor appears in a step but is not listed at the top of the use case
- A listed actor never appears in the steps or extensions
- Actor names are inconsistent ("workspace owner" in the header, "admin" in step 4 — same person?)
- System actions are attributed to the wrong actor

---

### Dimension 6: Goal level fit

Ask: "Is this use case at the right goal level?"

Flags:
- Too broad (summary level): the title needs "and"; the scenario has more than 10 steps; it spans multiple sessions or days; multiple distinct user goals are present → recommend splitting
- Too narrow (subfunction level): the scenario has fewer than 3 steps; the goal is a supporting action inside a larger flow; there is no meaningful precondition → recommend elevating or nesting inside a parent use case
- Correct level: 4–8 steps, one clear user goal, achievable in one sitting

---

## Output format

For each use case reviewed, output:

```
## Review: [Use Case ID] — [Title]

### Issues found

| # | Dimension | Location | Problem | Suggested resolution |
|---|---|---|---|---|
| 1 | Extension completeness | Step 3 | No extension for network failure | Add 3a: network timeout → show retry prompt |
| 2 | Precondition completeness | Preconditions | "User has editor role" — no extension if they don't | Add extension or note that the route is protected by role guard |
...

### Empty states implied
[List of preconditions whose absence is not handled anywhere — these require design]

### Summary
[One paragraph: overall quality, whether it is ready to use for generation, and the single most important issue to resolve first]
```

If no issues are found in a dimension, omit that dimension from the table. If a use case is clean across all dimensions, output: "No issues found. This use case is ready for downstream use."

## When to block downstream generation

Recommend blocking code or design generation if any of the following are true:
- A step is unimplementable without assumptions (Dimension 1 critical issue)
- More than 30% of steps have no extensions (Dimension 2)
- A precondition's absence would cause a runtime error with no handler (Dimension 3 critical issue)
- A postcondition contradicts an extension (Dimension 4 contradiction)

In these cases, output: "**Not ready for generation.** Resolve the issues above before proceeding."
