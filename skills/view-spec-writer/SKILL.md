---
name: view-spec-writer
description: >
  Use this skill when the user wants to specify a read-heavy feature that does not fit the fully dressed use case format. Triggers include 'spec this dashboard', 'define this search results view', 'write a view spec', 'specify the reporting page', 'document this list view', 'what should this feed display', or any request to formalise what a read-only screen shows. Also use when the user describes a filtered table, notification feed, analytics panel, admin list, or any feature where the primary interaction is viewing data rather than mutating it. Do NOT use for features where the user performs a meaningful action that changes system state — those are use cases and belong in use-case-writer.
---

# View Spec Writer

Produces view specifications for read-heavy features that do not fit the fully dressed use case format.

## Overview

Most products are 70% reads. Dashboards, search results, reporting views, filtered tables, notification feeds — these do not have a meaningful main success scenario beyond "user opens the page, system shows data." Forcing them into the fully dressed format produces either a trivially short spec or a grotesquely inflated one where the extensions are all display-logic edge cases.

View specifications capture what is shown, to whom, under what conditions, and with what performance expectations — without the actor-goal-scenario structure that use cases need.

## When to use this vs use-case-writer

| Signal | View spec | Use case |
|---|---|---|
| Primary interaction | Viewing, sorting, filtering, paginating | Creating, updating, deleting, submitting |
| System state change | None — read only | Yes — observable postcondition exists |
| Main success scenario length | Trivially short (1-3 steps) | 4-8 meaningful steps |
| Examples | Dashboard, search results, report, feed | Form submission, record creation, workflow transition |

**Grey area:** A view with inline editing needs both a view spec for the read layer and a use case for each mutation.

## Derivation process

### Step 1: Identify the view

Determine five fields: **View ID** (sequential: V-01, V-02), **Title** (user vocabulary, not technical — "Team activity feed" not "ActivityProjectionView"), **Actor** (role name, not persona), **Data source** (which aggregate or read model), **Trigger** (specific: a route, a navigation action, a tab switch).

### Step 2: Write display rules

Declarative, not procedural. Each rule covers one aspect. At minimum:
1. What data fields are shown and in what layout (table, list, cards, chart)
2. Default sorting, filtering, grouping, and pagination
3. What the user sees when the data set is empty

### Step 3: Write conditional display rules

How the view changes based on state, role, or data. Format: `Ca. [Condition] -> [what changes]`. Cover role-based visibility differences, data-dependent display changes (overdue items highlighted), and state-dependent changes (loading, error, partial data).

### Step 4: Define performance constraints

Target load time, pagination size, caching strategy. If unknown, write "TBD" rather than omitting the section.

## View spec template

```
View ID: V-XX
Title: [what the user sees]
Actor: [who sees it]
Data source: [which aggregate(s) or read model(s)]
Trigger: [what causes this view to appear — a route, a navigation action, a state change]

Display rules:
  1. [What is shown and how — declarative, not procedural]
  2. [Sorting, filtering, grouping, pagination defaults]
  3. [What happens when the data set is empty]

Conditional display:
  Ca. [Condition] → [what changes in the display]
  Cb. [Condition] → [what changes in the display]

Performance constraints:
  - [Target load time, pagination size, caching strategy if relevant]
```

## Integration with screen-inventory

View specs feed into `screen-inventory` alongside use cases. Screen IDs from view specs follow the pattern **S-V-[view number]-[sequence]** (e.g., S-V-01-01, S-V-01-02).

| View spec element | Screen inventory mapping |
|---|---|
| Display rules | Full screen or state entries |
| Conditional display rules | State variants of the parent screen |
| Empty data rule | Dedicated empty state screen |
| Loading / error conditions | Loading state and error state screens |

## Output and verification

Output view specs directly — no surrounding prose. Separate multiple specs with horizontal rules. After each, output this checklist:

```
Verification:
  [ ] Title uses user vocabulary, not technical names
  [ ] Actor is a role name, not a persona
  [ ] Data source references a known aggregate or read model
  [ ] Trigger is specific (a route, an action) not vague ("user navigates")
  [ ] At least three display rules: layout, defaults, empty state
  [ ] Conditional display covers: role differences, data-dependent changes, loading/error
  [ ] Performance constraints stated or marked TBD
  [ ] View ID is unique and sequential
```

## What view specs do NOT produce

View specs do not go through the TDD pipeline — they are tested with integration tests against the read model, not the four-level suite. They do not produce AI implementation prompts.

The downstream skill is `view-to-query-model`, which derives a read model specification from each view spec: source events, schema, indexes, and staleness tolerance.

## Common mistakes

- **Writing a use case as a view spec.** If the feature mutates system state, it is a use case.
- **Omitting the empty state.** Every view must define what happens when there is no data.
- **Vague triggers.** "User opens the page" is insufficient. "User navigates to /dashboard from the sidebar" is specific enough.
- **Procedural display rules.** "System queries the database and renders rows" is wrong. "Records displayed as a paginated table sorted by creation date descending" is correct.
- **Missing role-based conditional display.** If two actors see different data or controls, each needs its own conditional display rule or its own view spec.
