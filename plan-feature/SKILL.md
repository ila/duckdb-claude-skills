---
name: plan-feature
description: Structured feature planning with interrogation rounds and confirmation gate. Use when starting a non-trivial feature to align on requirements, scope, and edge cases before writing code.
---

# Feature Planning

Structured planning process for non-trivial features. Ensures requirements are clear, edge cases are considered, and there is explicit alignment before implementation begins.

## Process

### Step 1: Understand the request

Extract from the user's description:
- **What** they want (the feature)
- **Why** they want it (the problem it solves)
- **Who** benefits (end users, developers, CI)
- **Ambiguities** — anything unclear or unstated

### Step 2: Gather codebase context

Before asking questions:
1. Read CLAUDE.md for project conventions and constraints.
2. Identify which source files and subsystems are affected.
3. Search for existing patterns, utilities, or prior art that could be reused.
4. Note any related tests, docs, or configuration that will need updates.

### Step 3: Interrogation (up to 3 rounds)

Ask the user focused questions to resolve ambiguities. Each round addresses a different concern. Stop early if everything is clear.

**Round 1 — Requirements & scope:**
- What exactly should change? What should NOT change?
- Which components are affected?
- Are there existing patterns to follow?
- What is out of scope?

**Round 2 — Edge cases & error handling:**
- What happens with empty/NULL/boundary inputs?
- What happens if this interacts with feature X?
- Are there concurrent access concerns?
- What error behavior is expected?

**Round 3 — Acceptance criteria:**
- What tests prove this works? New test file or extend existing?
- What documentation needs updating?
- Any performance requirements or benchmarks?
- How should this be verified end-to-end?

### Step 4: Confirmation gate

Present a structured summary of the understood requirements:

```
## Feature: <name>
## Scope: <what changes, what doesn't>
## Affected files: <list>
## Edge cases: <list>
## Tests: <what to test, which file>
## Docs: <what to update>
## Out of scope: <list>
```

Get explicit user confirmation before proceeding to implementation.

### Step 5: Write the plan

After confirmation, produce a plan with:

1. **Files to modify** — each file with a description of changes
2. **New files** — if any, with purpose and outline
3. **Test strategy** — specific test scenarios
4. **Doc updates** — which doc files need changes
5. **Verification** — how to test end-to-end (build, run tests, manual check)

## DuckDB Extension Checklist

When planning features for DuckDB extensions, also consider:

- [ ] Which logical operator types are affected?
- [ ] Does this change any optimizer rewrite rule?
- [ ] Does this affect the parser extension?
- [ ] Are new DuckDB APIs needed? (Check the public API first)
- [ ] Does this need new extension settings (`AddExtensionOption`)?
- [ ] Does this need a new pragma function?
- [ ] Will this work with the existing test framework (SQLLogicTest)?
- [ ] Does the generated SQL change? (Cross-system impact)

## Anti-patterns

- **Do not plan in the abstract.** Every plan item must reference a specific file or function.
- **Do not plan features the user didn't ask for.** Scope creep kills projects.
- **Do not skip the confirmation gate.** Misaligned assumptions waste everyone's time.
- **Do not start coding during planning.** Separate thinking from doing.
