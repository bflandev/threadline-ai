---
name: use-case-writer
description: >
  Use this skill whenever the user wants to specify a feature, user flow, system behaviour, or product requirement as a structured use case. Triggers include 'write a use case', 'spec this feature', 'document this flow', 'define requirements for', 'turn this into a spec', or any request to formalise what a piece of software should do. Also use when the user describes a user journey, a product idea, or a set of user stories that need to be elevated into a structured specification. Do NOT use for writing code, generating tests, or producing design artefacts — those are downstream skills.
---

# Use Case Writer

Produces fully dressed use cases at the correct goal level from a description of what needs to be built.

## Overview

A use case is a structured description of how an actor achieves a goal by interacting with a system. The fully dressed format used here contains: primary actor, goal, preconditions, a numbered main success scenario, lettered extensions, and postconditions. This format is precise enough for AI code generation and review, while remaining readable by non-technical stakeholders.

## Step 1: Determine goal level

Before writing, classify the request into one of three goal levels. This controls scope and prevents under- or over-specified output.

| Level | Description | Trigger phrase examples | Output use |
|---|---|---|---|
| Summary | A business capability spanning multiple user goals | "manage a workspace", "handle billing", "run events" | Architecture prompts, journey maps |
| Sea level | A single user goal achievable in one sitting | "submit a form", "invite a collaborator", "publish a page" | Feature implementation, the default level |
| Subfunction | One step within a sea-level use case | "validate a required field", "resolve a relation field" | Individual function or component prompts |

**Default to sea level.** If the request is too broad for sea level (spans multiple sessions or multiple distinct goals), split it into multiple sea-level use cases. If it is too narrow (a single interaction within a larger flow), flag it as a subfunction and ask the user whether they want the parent sea-level use case written first.

## Step 2: Identify actors

List every actor who participates in the use case:

- **Primary actor** — the one who initiates the goal. Usually a human role (workspace owner, end user, editor) but can be a system in automated flows.
- **Supporting actors** — systems or people the primary actor interacts with (email service, payment gateway, another user role).
- **Stakeholders** — parties with an interest in the outcome who do not directly interact (auditor, organisation owner, regulator). Include these only when their interests constrain the system's behaviour.

Name actors by role, not by persona name. "Workspace editor" not "Sarah."

## Step 3: Write the use case

Use this exact template:

```
Use Case ID: UC-XX
Title: [verb + noun phrase — present tense, active voice]
Primary actor: [role name]
Goal: [one sentence — what the actor wants to achieve, not how]
Preconditions: [comma-separated list of what must be true before step 1]

Main success scenario:
  1. [Step — actor action or system response, declarative not procedural]
  2. ...

Extensions:
  Xa. [Condition at step X] → [what happens instead]
  Xb. ...

Postconditions: [comma-separated list of observable outcomes when scenario ends successfully]
```

## Step 4: Apply quality rules

Check each element against these rules before outputting.

### Title
- Verb + noun. "Submit event form" not "Event form submission."
- Present tense. "Create a table" not "Creating a table."

### Goal
- Describes the actor's intent, not the system mechanism. "Add a record to a shared table without seeing the table directly" not "POST to /records endpoint."

### Preconditions
- Each precondition is a binary assertion that is either true or false. "User has editor role" not "User is logged in and has been granted editor role by an admin who has configured the workspace."
- Preconditions that are not met define the empty states the UI must handle. Flag these explicitly: "If this precondition is not met, an empty state or onboarding gate is required."

### Main success scenario
- Steps are **declarative** (what happens) not **procedural** (how it happens). "System validates required fields" not "System iterates through the fields array and checks each required property."
- Each step has one clear subject. Not "User fills in the form and clicks submit."
- Alternate between actor and system steps naturally. Avoid long runs of pure system steps with no user interaction.
- Aim for 4–8 steps. Fewer than 4 suggests a subfunction, not a sea-level use case. More than 10 suggests it should be split.

### Extensions
- Every extension references the step it branches from. "4a." means the branch occurs at step 4.
- Use letters for multiple branches from the same step: 4a, 4b, 4c.
- Extensions cover: validation failures, missing preconditions, alternate success paths, system errors, and permission boundaries.
- **Do not omit extensions.** They are where most bugs and design gaps live. If you can think of it, write it.
- Extension format: `Xa. [condition] → [outcome]`. The condition is what is true; the outcome is what happens as a result.

### Postconditions
- Describe observable state after success. "A new record exists in the table" not "the database insert completed."
- Include postconditions for both the primary actor and any affected parties. "Submitter receives confirmation; owner can review the record."

## Step 5: Output format

Output use cases as a numbered list if there are multiple. Separate each with a horizontal rule. Do not wrap in prose — output the use case document directly.

If the user's request spans multiple goal levels, output:
1. A summary-level use case first (one paragraph, no numbered steps — just title, actors, goal, and a brief description of scope)
2. The sea-level use cases that decompose it

## Common mistakes to avoid

- **Writing the implementation in the steps.** "System calls the validateField() function" is wrong. "System validates required fields" is correct.
- **Missing extensions.** An error path that is not written is a design gap.
- **Vague preconditions.** "User is authenticated" is fine. "User has access" is not — access to what?
- **Postconditions that describe internal state.** "Database record is written" is internal. "A new event appears in the owner's table view" is observable.
- **One use case doing too much.** If the title needs "and", it should probably be two use cases.
