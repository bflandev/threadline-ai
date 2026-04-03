# Agentic Workflow Guide: Use Case Skill Suite

## What this guide covers

This guide explains how to run the seven use case skills as a connected agentic workflow — from a raw product description through to a developer-ready handoff package. It covers how the skills chain together, where human review fits, how to handle failures and loops, and how to prompt an AI agent to run the full pipeline.

---

## The seven skills

| Skill | Input | Output |
|---|---|---|
| `use-case-writer` | Product description or feature brief | Fully dressed use cases |
| `use-case-reviewer` | Use cases | Structured review with flagged issues |
| `screen-inventory` | Reviewed use cases | Screen inventory table |
| `excalidraw-generator` | Use cases + screen inventory | `.excalidraw` flow diagram files |
| `component-inventory` | Screen inventory | Component library with variants |
| `figma-spec` | Screen inventory + component inventory | Figma file specification document |
| `use-case-to-prompt` | Reviewed use cases | AI generation prompts (implementation, tests, models) |

---

## The pipeline

```
INPUT: product description / feature brief
        │
        ▼
┌─────────────────────┐
│  use-case-writer    │  → Produce fully dressed use cases at sea level
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│  use-case-reviewer  │  → Audit for ambiguities, missing extensions,
└─────────────────────┘    empty states, postcondition issues
        │
        ├─── Issues found? ──► HUMAN REVIEW → iterate use-case-writer
        │
        │    Clean? ──────────────────────────────────────┐
        ▼                                                  │
┌─────────────────────┐                                   │
│  screen-inventory   │  → Extract all screens and states │
└─────────────────────┘                                   │
        │                                                  │
        ▼                                                  │
┌─────────────────────────────────────────────────────┐   │
│  excalidraw-generator  (parallel with ↓)            │   │
│  → One .excalidraw file per use case                │   │
└─────────────────────────────────────────────────────┘   │
        │                                                  │
        ▼                                                  │
┌─────────────────────┐                                   │
│  component-inventory│  → Extract shared components      │
└─────────────────────┘    and their variants             │
        │                                                  │
        ▼                                                  │
┌─────────────────────┐                                   │
│  figma-spec         │  → Structured Figma file spec     │
└─────────────────────┘                                   │
        │                                                  │
        ▼                                                  │
 HUMAN REVIEW: Excalidraw + Figma spec                    │
        │                                                  │
        ├─── Design gaps found? ──► iterate use-case-writer (back to top)
        │                                                  │
        │    Approved? ◄────────────────────────────────────┘
        ▼
┌─────────────────────┐
│  use-case-to-prompt │  → Implementation prompt
└─────────────────────┘    Test generation prompt
                           Data model prompt
        │
        ▼
OUTPUT: handoff package
  • Use case documents
  • Excalidraw flow diagrams
  • Figma file specification
  • AI generation prompts (ready to paste into development environment)
```

---

## Running the pipeline: step by step

### Step 1: Invoke use-case-writer

Provide the agent with the product description and instruct it to read the `use-case-writer` SKILL.md before beginning. The agent should:

1. Classify the request by goal level
2. Identify all actors
3. Produce fully dressed use cases — one per distinct user goal
4. Flag any cases where goal level is ambiguous

**Prompt pattern:**

```
Read the use-case-writer SKILL.md then write use cases for the following:

[product description]

Produce sea-level use cases only. If a goal is too broad for a single use case, split it. 
Output each use case in the fully dressed format defined in the skill.
```

**What to check before moving on:**
- Every use case has a primary actor, goal, preconditions, numbered steps, extensions, and postconditions
- No use case title contains "and" (a sign it should be split)
- Extensions are present on every use case

---

### Step 2: Invoke use-case-reviewer

Pass all use cases from Step 1 to the reviewer. The agent should:

1. Review each use case across all six dimensions
2. Produce the structured review table
3. Explicitly state whether each use case is ready for downstream generation

**Prompt pattern:**

```
Read the use-case-reviewer SKILL.md then review the following use cases:

[paste use cases]

Produce a structured review for each. Explicitly state at the end of each review 
whether the use case is ready for downstream generation or not.
```

**Decision point: human review**

The agent outputs its review. A human reads it and decides:
- **All clean:** proceed to Step 3
- **Minor issues:** instruct the agent to fix them inline and re-review
- **Major issues:** return to Step 1 with the review as context

**Iteration prompt pattern (minor fixes):**

```
Apply these fixes to the use cases and re-run the reviewer:

[paste review issues]

Then confirm all use cases are ready.
```

Do not proceed past this step until `use-case-reviewer` outputs "ready for generation" for every use case.

---

### Step 3: Invoke screen-inventory

Pass all reviewed use cases to the screen inventory skill.

**Prompt pattern:**

```
Read the screen-inventory SKILL.md then extract the screen inventory 
from these use cases:

[paste reviewed use cases]

Output the full inventory table, the shared screens list, the empty states list, 
and the actor interface summary.
```

**What to check:**
- Every use case extension has at least one screen entry
- Every precondition has a corresponding empty state entry
- Screen IDs follow the pattern S-[UC number]-[sequence]

---

### Step 4: Invoke excalidraw-generator (can run in parallel with Steps 5–6)

The excalidraw-generator and component-inventory/figma-spec chains do not depend on each other. Run them in parallel if the agent supports concurrent tasks, or sequentially if not.

**Prompt pattern:**

```
Read the excalidraw-generator SKILL.md then generate one .excalidraw file 
for each of the following use cases. Write each file to disk named 
UC-XX-[title-slug].excalidraw.

[paste use cases and screen inventory]
```

The agent writes files to `/mnt/user-data/outputs/`. Confirm each file is written before proceeding.

---

### Step 5: Invoke component-inventory

**Prompt pattern:**

```
Read the component-inventory SKILL.md then derive the component library 
from the following screen inventory:

[paste screen inventory]

Output the component table and the variants table for each component. 
Flag all shared components and actor-specific variants.
```

**What to check:**
- Variant names are condition-based, not treatment-based
- Every extension state in the screen inventory has a corresponding component variant
- Shared components are identified

---

### Step 6: Invoke figma-spec

**Prompt pattern:**

```
Read the figma-spec SKILL.md then produce a Figma file specification 
from the following screen inventory and component inventory:

[paste screen inventory]
[paste component inventory]

Output all six sections of the spec. Include the complete handoff checklist.
```

---

### Step 7: Human review of design artefacts

At this point the human has:
- Excalidraw flow diagrams (open each at excalidraw.com)
- A Figma file specification

Review the Excalidraw diagrams first. Check:
- Every step in each use case is represented
- Every extension is shown as a branch
- No steps are missing, reordered, or combined

Review the Figma spec second. Check:
- Every screen in the screen inventory appears in the spec
- Every component variant appears in the spec
- The handoff checklist is complete

**If gaps are found:** return to Step 1 with the gap description. The use case was likely missing a step or extension. Fix the use case, re-run the reviewer, and cascade the change through the downstream steps.

**If approved:** proceed to Step 8.

---

### Step 8: Invoke use-case-to-prompt

**Prompt pattern:**

```
Read the use-case-to-prompt SKILL.md then generate all three prompt types 
(implementation, test, data model) for each of the following use cases:

[paste reviewed use cases]

Framework: [language/framework]
Test framework: [test library]
ORM/database: [data layer]
Existing context: [any relevant existing code, models, or conventions]

Chain the prompts in the correct order: data model → implementation → tests.
```

The agent produces three prompts per use case, ready to paste into a development environment or a code-generation agent.

---

## Handling the full pipeline with a single prompt

If the agent supports multi-step agentic execution, you can initiate the entire pipeline with one prompt. The agent reads each skill in sequence and gates on the reviewer before proceeding.

**Single-prompt pipeline initiation:**

```
You are running a multi-step agentic pipeline. Read each SKILL.md file 
as instructed at each step. Do not proceed past use-case-reviewer until 
all use cases are confirmed ready.

Skills are located at:
  /path/to/skills/use-case-writer/SKILL.md
  /path/to/skills/use-case-reviewer/SKILL.md
  /path/to/skills/screen-inventory/SKILL.md
  /path/to/skills/excalidraw-generator/SKILL.md
  /path/to/skills/component-inventory/SKILL.md
  /path/to/skills/figma-spec/SKILL.md
  /path/to/skills/use-case-to-prompt/SKILL.md

Pipeline:
  1. use-case-writer → produce use cases
  2. use-case-reviewer → review all. If issues found, fix and re-review before step 3.
  3. screen-inventory → produce screen inventory
  4. excalidraw-generator → write one .excalidraw file per use case to disk
  5. component-inventory → produce component inventory
  6. figma-spec → produce Figma spec document
  7. use-case-to-prompt → produce all three prompt types per use case

Pause after step 4 and ask for human approval of the Excalidraw diagrams 
before continuing to steps 5–7.

Input:
[product description]

Framework context:
[language, framework, test library, ORM]
```

---

## Where human review must happen

The pipeline has two mandatory human review gates. Do not automate past these.

### Gate 1: After use-case-reviewer (Step 2)

The reviewer flags issues, but a human decides whether they are blocking. Some reviewer issues are structural (wrong goal level, missing extensions) and must be fixed. Others are informational (an empty state exists but the design will handle it without a use case change) and can be acknowledged and skipped.

The agent cannot make this judgement — it does not know what design decisions the team has already made.

### Gate 2: After excalidraw-generator (Step 4)

The Excalidraw diagrams make the use cases spatial and discussable. Stakeholders who would not read a use case document will engage with a swim-lane diagram. Issues caught here (missing steps, wrong actor assignments, missing extensions) are much cheaper to fix than issues caught in Figma or in code.

This gate should involve at minimum: one person who wrote the use cases, and one person who will implement the feature.

---

## Iterating on a single use case

When requirements change after the pipeline has run, do not re-run the full pipeline. Target the affected use case only:

1. Edit the use case
2. Re-run `use-case-reviewer` on the changed use case
3. Update the screen inventory for the affected use case only (add, remove, or rename affected Screen IDs)
4. Regenerate the Excalidraw file for the affected use case
5. Update the component inventory if new variants are required
6. Update the Figma spec for the affected use case page
7. Regenerate the implementation and test prompts for the affected use case

Do not regenerate unaffected use cases — their Screen IDs, component variants, and prompts remain valid.

---

## File outputs

The pipeline produces these files:

```
outputs/
  use-cases.md                          ← all use cases (Steps 1–2)
  screen-inventory.md                   ← screen inventory table (Step 3)
  UC-01-[slug].excalidraw               ← flow diagram (Step 4)
  UC-02-[slug].excalidraw
  ...
  component-inventory.md                ← component library (Step 5)
  figma-spec.md                         ← Figma file specification (Step 6)
  prompts/
    UC-01-data-model-prompt.md          ← (Step 8)
    UC-01-implementation-prompt.md
    UC-01-test-prompt.md
    UC-02-data-model-prompt.md
    ...
```

Keep all files in version control alongside the codebase. When a use case changes, the downstream files change with it — version control makes this traceable.

---

## Failure modes and how to handle them

**use-case-writer produces a single giant use case**
The agent bundled multiple goals into one. Return with: "Split this into separate use cases — one per distinct user goal. Each use case title should be achievable in one sitting."

**use-case-reviewer finds no issues on a complex use case**
The agent is being too lenient. Return with: "Re-review this use case and specifically ask: for every step, what can go wrong? For every precondition, what happens if it is not met?"

**excalidraw-generator produces a file that doesn't open**
The JSON is malformed. Return with: "Validate the JSON before writing. Check that every element has a unique ID, that arrow points arrays have exactly two entries, and that the top-level structure matches the Excalidraw format exactly."

**component-inventory uses treatment-based variant names**
Return with: "Rename all variants to describe the condition that produces them, not the visual treatment. 'required-error' not 'red-outline'. 'restricted' not 'grayed-out'."

**use-case-to-prompt generates tests without step references in the names**
Return with: "Every test name must follow the pattern test_uc[ID]_step[N]_[description] or test_uc[ID]_ext[Na]_[description]. Rename all tests that do not follow this pattern."

---

## The handoff package

At the end of the pipeline, the developer receives:

1. **Use case documents** — the spec and source of truth
2. **Excalidraw files** — the navigation model, one per use case
3. **Figma file specification** — the design blueprint
4. **Generation prompts** — ready to run in any AI coding environment

The chain of traceability runs in both directions:

- **Forward** (spec to code): use case step → screen ID → Figma frame → component variant → implementation prompt → test name
- **Backward** (bug to spec): failing test name → use case extension → use case step → design decision

This chain is the value of the process. Any break in it — a screen without an annotation, a test without a step reference, a component variant named after a colour — is a gap in traceability and should be treated as a defect in the process, not just a style issue.
