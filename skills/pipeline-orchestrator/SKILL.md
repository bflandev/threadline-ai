---
name: pipeline-orchestrator
description: >
  Use this skill when the user wants to run the complete use-case-driven development pipeline from a product description through to a developer-ready handoff package including TDD tests, technical design, Excalidraw diagrams, and Figma spec. Triggers include 'run the full pipeline', 'take this from spec to code', 'run all the skills', 'full pipeline from this description', 'end to end from use cases', 'orchestrate the pipeline'. This is the meta-skill — it reads and invokes all other skills in the correct order. Always read this SKILL.md first, then read each sub-skill's SKILL.md before invoking it. Do NOT skip the use-case-reviewer gate. Do NOT proceed to implementation prompts before the human review gate is cleared.
---

# Pipeline Orchestrator

The meta-skill. Reads and invokes all other skills in the correct sequence, manages human review gates, runs parallel branches, and produces the complete artifact set.

## Overview

This skill coordinates nine sub-skills into a single coherent pipeline. It is responsible for:
- Reading each sub-skill's SKILL.md at the point of invocation
- Passing the correct inputs and outputs between skills
- Enforcing the two mandatory human review gates
- Running parallel branches where the DAG allows it
- Producing the artifact manifest at the end

## Sub-skills in this pipeline

| Skill | SKILL.md location | Role |
|---|---|---|
| `use-case-writer` | skills/use-case-writer/SKILL.md | Spec |
| `use-case-reviewer` | skills/use-case-reviewer/SKILL.md | Quality gate |
| `use-case-to-tdd` | skills/use-case-to-tdd/SKILL.md | Tests first |
| `use-case-to-technical-design` | skills/use-case-to-technical-design/SKILL.md | Architecture |
| `screen-inventory` | skills/screen-inventory/SKILL.md | Design input |
| `excalidraw-generator` | skills/excalidraw-generator/SKILL.md | Flow diagrams |
| `component-inventory` | skills/component-inventory/SKILL.md | Component library |
| `figma-spec` | skills/figma-spec/SKILL.md | Design spec |
| `use-case-to-prompt` | skills/use-case-to-prompt/SKILL.md | AI generation prompts |

## The Pipeline DAG

```
INPUT
  │
  ▼
[1] use-case-writer
  │
  ▼
[2] use-case-reviewer  ◄──────── GATE 1: human review
  │                              Fix issues → back to [1]
  │  (all use cases ready)
  │
  ├──────────────────────────────────────────────────────┐
  │                                                      │
  ▼                                                      ▼
[3a] use-case-to-tdd                          [3b] screen-inventory
  │  (acceptance tests,                            │
  │   domain unit tests,                           ├──► [4a] excalidraw-generator
  │   interactor tests,                            │
  │   integration tests)                           └──► [4b] component-inventory
  │                                                           │
  ▼                                                           ▼
[3c] use-case-to-technical-design                       [4c] figma-spec
  │  (bounded contexts,
  │   aggregates, ports,
  │   adapters, directory
  │   structure)
  │
  └────────────────────────────────────────────────────┐
                                                        │
                          GATE 2: human review          │
                          (Excalidraw + Figma spec       │
                           + technical design)    ◄──────┘
                                │
                                │ (approved)
                                ▼
                    [5] use-case-to-prompt
                        (implementation prompts,
                         test prompts,
                         data model prompts)
                                │
                                ▼
                            OUTPUT
```

## Execution procedure

### Phase 0: Read this file and plan

Before doing anything else:
1. Confirm the input is a product description or feature brief
2. Confirm the target language(s) (TypeScript / GoLang / both)
3. Confirm the framework context if available
4. Announce the pipeline plan to the user: "I'll run the pipeline in these phases: [list]. I'll pause twice for your review."

### Phase 1: Specification — use-case-writer

**Read:** `skills/use-case-writer/SKILL.md`

**Inputs:** product description, any actor or domain context the user provides

**Execute:**
1. Classify goal level — if the description is summary level, decompose into sea-level use cases
2. Write all sea-level use cases in the fully dressed format
3. Number them sequentially: UC-01, UC-02, etc.

**Output:** `use-cases.md` — all use cases in a single file

**Transition check:** Count the use cases. If any title contains "and", flag it as a candidate for splitting before proceeding.

---

### Phase 2: Review — use-case-reviewer [GATE 1]

**Read:** `skills/use-case-reviewer/SKILL.md`

**Inputs:** `use-cases.md` from Phase 1

**Execute:**
1. Review every use case across all six dimensions
2. Produce the structured review table
3. Explicitly mark each use case: READY or NOT READY

**Gate behaviour:**
- If ALL use cases are READY → proceed automatically to Phase 3
- If ANY use case is NOT READY → STOP. Present the review to the user. Wait for instruction: fix inline or return to Phase 1 with the review as context.
- Do not proceed past this gate without explicit confirmation that all use cases are ready.

**Output:** Review report (inline, not a file). Updated `use-cases.md` if fixes were applied.

---

### Phase 3: Parallel branch — TDD + Technical Design + Design

Phase 3 has three parallel branches. Run them concurrently if the execution environment supports it, sequentially otherwise. All three receive the same input: the reviewed, ready use cases.

#### Branch A: use-case-to-tdd

**Read:** `skills/use-case-to-tdd/SKILL.md`

**Inputs:** reviewed use cases, target language

**Execute:** Produce all four test levels for every use case:
1. Gherkin acceptance scenarios
2. Domain unit test skeletons
3. Interactor test skeletons with mocked ports
4. Integration/contract test skeletons

**Output files:**
```
tests/
  acceptance/UC-XX-[slug].acceptance.test.ts   (one per use case)
  unit/domain/[context]/[Aggregate].test.ts    (one per aggregate)
  unit/application/[context]/[UseCase].test.ts (one per use case)
  integration/adapters/[adapter]/[Repo].test.ts
```

**Critical:** All tests must be written to be RED. No implementation yet. If the user asks "but these will all fail" — that is correct. That is the point.

#### Branch B: use-case-to-technical-design

**Read:** `skills/use-case-to-technical-design/SKILL.md`

**Inputs:** reviewed use cases, target language, any existing codebase context

**Execute:** Produce the full technical design:
1. Bounded context map
2. Aggregate designs with invariants, typed errors, domain events
3. Use case interactor code
4. Port interface definitions
5. Adapter sketches (one driving, one driven per context)
6. Directory structure
7. Contract test skeletons

**Output files:**
```
technical-design/
  bounded-contexts.md
  domain/[context]/[Aggregate].[ts|go]
  application/[context]/[UseCase]UseCase.[ts|go]
  application/[context]/ports.[ts|go]
  adapters/http/[UseCase]Controller.[ts|go]
  adapters/[db]/[Aggregate]Repository.[ts|go]
  directory-structure.md
```

#### Branch C: Design pipeline (screen-inventory → excalidraw → components → figma)

**Read in sequence:** screen-inventory, excalidraw-generator, component-inventory, figma-spec SKILL.md files

**Inputs:** reviewed use cases

**Execute in sequence:**
1. `screen-inventory` → `screen-inventory.md`
2. `excalidraw-generator` → one `.excalidraw` file per use case
3. `component-inventory` → `component-inventory.md`
4. `figma-spec` → `figma-spec.md`

**Output files:**
```
design/
  screen-inventory.md
  UC-01-[slug].excalidraw
  UC-02-[slug].excalidraw
  ...
  component-inventory.md
  figma-spec.md
```

---

### Phase 4: Human review [GATE 2]

**Present to the user:**
1. The Excalidraw files: "Open each at excalidraw.com — verify every step and extension is present"
2. The technical design: "Review bounded contexts, aggregate invariants, and directory structure"
3. The TDD test skeletons: "Verify every extension has a test, every postcondition has an assertion"
4. The Figma spec: "Review page structure, component variants, annotation conventions"

**Gate behaviour:**
- If APPROVED → proceed to Phase 5
- If GAPS FOUND in use cases → return to Phase 1. Cascade changes through Phases 2 and 3.
- If DESIGN GAPS only → update the affected design files without re-running Phase 1 or 2
- If TECHNICAL GAPS only → update the technical design files without re-running the design branch

Do not proceed to Phase 5 without explicit user approval.

---

### Phase 5: Generation prompts — use-case-to-prompt

**Read:** `skills/use-case-to-prompt/SKILL.md`

**Inputs:** reviewed use cases, framework context, TDD test file names (for test context)

**Execute:** For each use case, produce three prompts:
1. Data model prompt (run first — establishes schema)
2. Implementation prompt (references schema + test file names)
3. Test implementation prompt (completes the TDD skeletons)

**The implementation prompt must reference the TDD test files** so the AI generates code that makes those specific tests pass, not hypothetical tests.

**Output files:**
```
prompts/
  UC-XX-data-model.md
  UC-XX-implementation.md
  UC-XX-tests.md
```

---

### Phase 6: Artifact manifest

At the end of the pipeline, produce a complete manifest of all files created:

```markdown
## Pipeline Artifact Manifest

### Specification
- [ ] use-cases.md

### TDD Test Suites
- [ ] tests/acceptance/UC-XX-[slug].acceptance.test.ts  (× use case count)
- [ ] tests/unit/domain/[context]/[Aggregate].test.ts   (× aggregate count)
- [ ] tests/unit/application/[context]/[UseCase].test.ts (× use case count)
- [ ] tests/integration/adapters/...

### Technical Design
- [ ] technical-design/bounded-contexts.md
- [ ] technical-design/domain/... (× aggregate count)
- [ ] technical-design/application/... (× use case count)
- [ ] technical-design/adapters/...
- [ ] technical-design/directory-structure.md

### Design Artefacts
- [ ] design/screen-inventory.md
- [ ] design/UC-XX-[slug].excalidraw (× use case count)
- [ ] design/component-inventory.md
- [ ] design/figma-spec.md

### Generation Prompts
- [ ] prompts/UC-XX-data-model.md (× use case count)
- [ ] prompts/UC-XX-implementation.md (× use case count)
- [ ] prompts/UC-XX-tests.md (× use case count)
```

Mark each item as complete. Flag any missing items with the phase that should have produced them.

---

## State management between phases

The orchestrator maintains a pipeline state object. Update it after each phase:

```
PIPELINE STATE
  use_cases: [list of UC-IDs]
  review_status: { UC-01: READY, UC-02: READY, ... }
  gate_1_cleared: true/false
  gate_2_cleared: true/false
  artifacts: { [filename]: complete/missing }
  current_phase: 1–6
  language: typescript | golang | both
  framework_context: { ... }
```

If the conversation is interrupted, reconstruct state from the artifacts already produced before resuming.

---

## Handling partial runs

If the user wants to run only part of the pipeline:

- "Just write the use cases" → Phase 1 only
- "Just the tests" → Phase 1 → 2 → 3A only
- "Just the design artefacts" → Phase 1 → 2 → 3C only
- "Just the technical design" → Phase 1 → 2 → 3B only
- "Skip to the prompts" → requires gate 2 cleared — if not, refuse and explain why

The gates are not optional regardless of which branches are run. If use-case-reviewer has not cleared a use case, no downstream skill runs against it.

---

## Error handling

**use-case-writer produces overlapping use cases:** Flag the overlap, ask user which goal takes precedence, split or merge accordingly.

**use-case-reviewer blocks on a use case after two fix attempts:** Escalate to user with: "After two revision passes, UC-XX still has unresolved issues: [list]. Please resolve these manually before we continue."

**excalidraw-generator produces malformed JSON:** Re-run with explicit instruction to validate JSON structure before writing. If it fails again, produce the diagram elements as a description the user can enter manually.

**Parallel branches produce conflicting outputs:** The screen inventory and technical design must agree on aggregate names and context boundaries. If they diverge, surface the conflict and ask the user to decide before producing the Figma spec or generation prompts.

**Gate 2 reveals use case gaps:** If a screen exists in the inventory that has no corresponding step or extension in the use cases, that step or extension is missing. Return to Phase 1 — do not patch the screen inventory to paper over the gap.

---

## Invocation prompt template

Use this to initiate the full pipeline:

```
Read skills/pipeline-orchestrator/SKILL.md and run the complete pipeline.

Product description:
[description]

Language(s): [TypeScript | GoLang | both]

Framework context:
[language, HTTP framework, ORM, test library, existing patterns]

Available skill files:
  skills/use-case-writer/SKILL.md
  skills/use-case-reviewer/SKILL.md
  skills/use-case-to-tdd/SKILL.md
  skills/use-case-to-technical-design/SKILL.md
  skills/screen-inventory/SKILL.md
  skills/excalidraw-generator/SKILL.md
  skills/component-inventory/SKILL.md
  skills/figma-spec/SKILL.md
  skills/use-case-to-prompt/SKILL.md

Pause at both human review gates. Do not proceed past either gate without
explicit confirmation. Present all artifacts from each phase before pausing.
```
