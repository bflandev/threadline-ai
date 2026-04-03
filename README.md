# Threadline

A Claude Code plugin for use-case-driven software development. A single specification artifact — the fully dressed use case — drives every downstream activity: product design, system architecture, test-driven development, and AI-assisted code generation.

Every test name traces to a use case ID. Every screen traces to a step. Every line of code traces to a test. When a bug surfaces, you follow the thread backward from code to requirement in seconds.

## The Pipeline

Threadline coordinates 16 skills through a directed acyclic graph with two human review gates:

```
[Product description]
        |
  use-case-writer + quality-attribute-writer        Phase 1: Specify
        |
  use-case-reviewer                                 Phase 2: Review (Gate 1)
        |
   +---------+-----------------+----------------+
   |         |                 |                |
 3A: TDD  3B: Architecture  3C: Design       3D: Views
   |         |                 |                |
   |    technical-design   screen-inventory  view-spec-writer
   |    api-contract-      excalidraw-gen    view-to-query-
   |     writer            component-inv      model
   |         |             figma-spec          |
   |         |             component-to-code   |
   +---------+-----------------+----------------+
        |
  Human review                                      Gate 2
        |
  use-case-to-prompt                                Phase 5: Generate
        |
  [Developer-ready handoff package]
```

All four Phase 3 branches run in parallel. Tests are written RED — no implementation until Gate 2 clears.

## Installation

```bash
# Local testing
claude --plugin-dir /path/to/threadline-ai

# Install for personal use
claude plugin install threadline-ai --scope user

# Install for a project (shared with team)
claude plugin install threadline-ai --scope project
```

Reload during development:

```
/reload-plugins
```

## Quick Start

**Full pipeline** — run all 16 skills end to end:

```
/threadline-ai:pipeline-orchestrator
```

Or invoke the pipeline-runner agent, which coordinates the full DAG with gate pauses:

```
Use the threadline-ai:pipeline-runner agent to run the full pipeline.

Product description: [your feature]
Language: TypeScript
Framework: Express + Prisma + Jest
```

**Threadline Lite** — for APIs, backend services, or incremental adoption:

```
Phase 1: /threadline-ai:use-case-writer
Phase 2: /threadline-ai:use-case-reviewer
Phase 3: /threadline-ai:use-case-to-tdd
Phase 4: /threadline-ai:use-case-to-prompt
```

**Individual skills** — run any skill standalone:

```
/threadline-ai:use-case-writer
```

## Skills Reference

### Specification (Phase 1-2)

| Skill | Purpose |
|-------|---------|
| `use-case-writer` | Write fully dressed use cases from product descriptions |
| `use-case-reviewer` | Audit use cases across six dimensions (Gate 1) |
| `quality-attribute-writer` | Specify non-functional requirements (performance, security, compliance) |

### Architecture (Phase 3B)

| Skill | Purpose |
|-------|---------|
| `use-case-to-technical-design` | Derive Clean Architecture + DDD from use cases |
| `api-contract-writer` | Derive REST/GraphQL endpoint specs from use cases + typed errors |

### Testing (Phase 3A)

| Skill | Purpose |
|-------|---------|
| `use-case-to-tdd` | Derive four-level TDD test suites (acceptance, domain, interactor, integration) |

### Design (Phase 3C)

| Skill | Purpose |
|-------|---------|
| `screen-inventory` | Extract screens, states, and UI moments from use cases |
| `excalidraw-generator` | Produce `.excalidraw` flow diagrams with swim lanes |
| `component-inventory` | Derive reusable UI component library from screens |
| `figma-spec` | Produce Figma file specification for designers |
| `component-to-code` | Derive framework-specific component specs (React, Vue, Svelte) |

### Views (Phase 3D)

| Skill | Purpose |
|-------|---------|
| `view-spec-writer` | Specify read-heavy features (dashboards, tables, reports) |
| `view-to-query-model` | Derive CQRS read models from view specs + domain events |

### Generation (Phase 5)

| Skill | Purpose |
|-------|---------|
| `use-case-to-prompt` | Produce AI code-generation prompts constrained by pre-existing tests |

### Operations

| Skill | Purpose |
|-------|---------|
| `pipeline-orchestrator` | Coordinate all skills through the DAG with gate enforcement |
| `threadline-audit` | Detect drift between use cases, tests, screens, design, and code |

### Standalone

| Skill | Purpose |
|-------|---------|
| `agentic-patterns` | Design multi-agent systems using 12 proven patterns |

## Agents

| Agent | Purpose |
|-------|---------|
| `pipeline-runner` | Runs the full or Lite pipeline autonomously with gate pauses |
| `threadline-auditor` | Runs drift detection and produces audit or change impact reports |

## Pipeline Modes

| Mode | Skills | Best for |
|------|--------|----------|
| **Full** | All 16 | Products with UI, teams with designers |
| **Lite** | 4 (writer, reviewer, tdd, prompt) | APIs, backend services, solo developers |

Lite artifacts are fully compatible with the Full pipeline. Add skills incrementally as the team grows.

## Methodology

The full methodology is documented in:

- **[Threadline Master Guide](docs/threadline-master-guide.md)** — complete v1 reference covering the use case format, mechanical derivation, Clean Architecture mapping, four-level TDD, and the eleven principles
- **[TDD + Clean Architecture Guide](docs/tech-guides/tdd-clean-architecture-pipeline-guide.md)** — deep dive on the double loop, the two TDD schools reconciled, mapping use case elements to tests, and code examples in TypeScript and Go
- **[v2 Gaps and Roadmap](docs/threadline-v2-gaps-and-roadmap.md)** — honest assessment of methodology gaps, proposed solutions (view specs, quality attributes, API contracts, drift detection), and the Threadline Lite specification

### Core Ideas

**The mechanical derivation.** Preconditions become invariant guards. Extensions become typed errors. Postconditions become test assertions. MSS steps become interactor method bodies. Same use case, same architecture, every time.

**The double loop.** An outer BDD acceptance test defines "done" for the use case. Inner unit and integration tests, one per architectural layer, collectively make the outer test go green.

**TDD-aware AI prompts.** The instruction is "make these pre-existing tests pass" — not "generate tests and code together." The tests are derived from the use case. The code must satisfy the tests. The use case is enforced transitively.

**Bidirectional traceability.** Test names reference use case IDs. Screen IDs reference step numbers. Figma annotations reference extension IDs. Follow the thread forward from requirement to code, or backward from bug to requirement.

## License

MIT
