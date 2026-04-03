---
name: quality-attribute-writer
description: >
  Use this skill when the user wants to specify non-functional requirements — performance, security, scalability, availability, compliance, or observability. Triggers include 'what are the NFRs', 'add performance requirements', 'specify security constraints', 'define SLAs', 'write quality attributes', or any request to formalise how well the system must behave. Also use when use cases mention stakeholder concerns about load, uptime, audit, or regulation. Do NOT use for functional behaviour (use-case-writer) or architectural decisions (use-case-to-technical-design).
---

# Quality Attribute Writer

Produces structured quality attribute scenarios that attach non-functional requirements to specific use cases or to the system as a whole.

## Overview

Use cases specify what the system must do. Quality attribute scenarios specify how well. Without structured specification, non-functional decisions are made ad hoc at the adapter layer — inconsistent, undocumented, untestable. QA scenarios make every NFR traceable, measurable, and verifiable using the SEI quality attribute scenario format.

## Step 1: Identify quality concerns

Scan these sources:

- **Stakeholder interests** — "compliance requires change tracking", "submitter expects sub-second response"
- **Actor declarations** — public actors imply rate limiting; authenticated actors imply session security
- **System constraints** — stated SLAs, regulatory requirements, expected user volume
- **Extension patterns** — high-frequency extensions signal performance concerns; permission extensions signal security concerns

## Step 2: Classify by attribute

Assign each candidate to exactly one attribute. If a concern spans two, write two scenarios.

| Attribute | Concerns | Example stimulus |
|---|---|---|
| Performance | Response time, throughput, latency | 1000 concurrent form submissions |
| Security | Auth, input validation, data protection | User attempts SQL injection |
| Scalability | Load growth, data volume, multi-region | User base grows 1K to 100K in 6 months |
| Availability | Uptime SLA, failover, degradation | Primary database becomes unreachable |
| Compliance | GDPR, SOC 2, HIPAA, audit, retention | Auditor requests 90-day access log |
| Observability | Logging, tracing, metrics, alerting | Error rate exceeds 1% over 5 minutes |

## Step 3: Write the scenario

```
QA-XX: [Short name]
Applies to: UC-XX (or "system-wide")
Attribute: [performance | security | scalability | availability | compliance | observability]
Stimulus: [What triggers the concern]
Response: [What the system must do]
Measure: [How to verify]
```

### Field rules

- **QA-XX** — sequential from QA-01. IDs propagate into technical design, integration tests, and adapter decisions.
- **Applies to** — a UC-ID, or "system-wide" only when genuinely universal.
- **Stimulus** — concrete and quantified. "High load" is wrong. "1000 concurrent POSTs within 10 seconds" is correct.
- **Response** — what, not how. "Process within 200ms p95" not "Use Redis caching."
- **Measure** — executable verification. "k6 load test at 1000 VUs" or "OWASP ZAP scan passes."

## Attribute-to-architecture mapping

QA scenarios feed into adapter and infrastructure layers. `use-case-to-technical-design` annotates choices with the QA-ID that requires them.

| Attribute | Architectural decisions influenced |
|---|---|
| Performance | Caching, connection pooling, query optimisation, CDN, async processing |
| Security | Auth middleware, input sanitisation, encryption, CORS, rate limiting |
| Scalability | Horizontal scaling, partitioning, queue decoupling, read replicas |
| Availability | Health checks, circuit breakers, retry policies, multi-AZ deployment |
| Compliance | Audit event handlers, retention policies, PII encryption, consent tracking |
| Observability | Structured logging, distributed tracing, metric exporters, alerting rules |

## Pipeline integration

QA scenarios are produced in **Phase 1** alongside use cases, reviewed at **Gate 2** alongside technical design (not Gate 1).

```
Phase 1: use-case-writer + quality-attribute-writer (parallel)
Phase 2: use-case-reviewer (Gate 1 — use cases only)
Phase 3B: use-case-to-technical-design (consumes QA scenarios)
Gate 2: QA scenarios reviewed alongside technical design
```

Integration tests gain a new category: **performance and security contract tests**, named `QA-XX_[attribute]_[short_description]`.

## Output format

Output all scenarios in `quality-attributes.md` with a summary table at the top:

```
## Quality Attribute Summary

| ID | Short name | Attribute | Applies to |
|---|---|---|---|
| QA-01 | Form submission throughput | Performance | UC-02 |
| QA-02 | Public endpoint injection defence | Security | UC-02 |

---

QA-01: Form submission throughput
Applies to: UC-02
Attribute: performance
Stimulus: 1000 concurrent form submissions within 10 seconds
Response: Process each within 200ms p95, zero data loss
Measure: k6 load test — 1000 VUs, 60s, p95 < 200ms, 0 errors

---
...
```

### Verification checklist

- [ ] Every stimulus is quantified (no "high load" or "many users")
- [ ] Every response describes what, not how (no implementation decisions)
- [ ] Every measure is executable (a tool, a test, an audit procedure)
- [ ] QA-IDs are sequential with no gaps
- [ ] Every UC-ID in "Applies to" exists in the use case set
- [ ] No duplicate scenarios for the same concern on the same use case
- [ ] System-wide scenarios genuinely apply everywhere, not just most use cases

## Common mistakes

- **Vague stimuli.** "System is under heavy load" is a worry, not a scenario. Quantify: users, payloads, time window.
- **Architecture in the response.** "Use Redis" belongs in technical design. Write "return cached results within 50ms" instead.
- **Missing the measure.** A scenario without a measure is a wish, not a requirement.
- **Over-applying system-wide.** Rate limiting may apply to public endpoints only. Audit logging to writes only. Be specific.
- **Ignoring observability.** Without it, performance and security cannot be verified in production. Pair performance scenarios with observability scenarios.
