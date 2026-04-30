---
name: code-review
description: Multi-perspective code review of the most recent commit. Launches 5 parallel review agents covering correctness, reliability, performance, simplicity, and style. Reports findings with severity and confidence scores.
---

# Code Review

Perform a thorough, multi-perspective code review of the most recent commit (`git show HEAD`).

## Process

### Step 1: Inspect the change

```bash
git show HEAD
```

Read every line of the diff and commit message carefully.

### Step 2: Gather context

Before launching review agents, gather all context they will need:

1. **Read all modified files in full** — not just the diff. Understand the surrounding code.
2. **Find callers and dependencies** — use Grep/Glob to find callers of modified functions, related interfaces, and types.
3. **Check the commit stack** — `git log --oneline -10` for recent context.
4. **Read AGENTS.md or CLAUDE.md** for project-specific rules the change must follow.

### Step 3: Launch 5 review agents in parallel

Each agent reviews from a specific perspective. Provide each agent with:
- The full diff
- The full content of modified files
- Relevant caller/dependency excerpts
- Project conventions from AGENTS.md or CLAUDE.md

#### Perspective 1: Correctness

- Is the algorithm correct? Are there off-by-one errors, integer overflow risks?
- Does the code do what the commit message claims?
- Are edge cases handled? NULL inputs, empty collections, boundary values, maximum sizes?
- For DuckDB extensions: are logical plan transformations semantics-preserving? Are column bindings correct after rewrites?

#### Perspective 2: Reliability

- Are there tests? Do they cover edge cases and error paths?
- Could the tests pass while the code is still broken? (e.g., too-wide thresholds, missing assertions)
- Are errors handled or silently swallowed?
- What happens if this code fails mid-execution? Is state left consistent?
- For DuckDB extensions: are SQL test cross-checks bidirectional (`EXCEPT ALL` both ways)?

#### Perspective 3: Performance

- Are there allocations in hot paths? Unnecessary copies or clones?
- Could this cause memory pressure or unbounded growth?
- Are there O(n²) or worse algorithms that could be O(n) or O(n log n)?
- For DuckDB extensions: does the change affect query plan compilation time? Are there unnecessary plan tree traversals?

#### Perspective 4: Simplicity

- Is there dead code, unused imports, or unreachable branches?
- Are there abstractions with only one implementation that add indirection without value?
- Is there speculative "just in case" code that handles scenarios that cannot occur?
- Could verbose patterns be replaced with idiomatic equivalents?
- Is there excessive nesting that could be flattened with early returns?

#### Perspective 5: Style & conventions

- Does the code follow the project's AGENTS.md or CLAUDE.md rules?
- Are naming conventions followed? (CamelCase functions, lower_case variables, UPPER_CASE constants)
- Are debug print statements added at major flow points?
- Does this change need documentation updates? New tests?
- Is the commit message accurate and complete?

### Step 4: Collect and categorize findings

Each agent reports findings with:
- **Severity**: CRITICAL / MAJOR / MINOR / NIT
- **Confidence**: 0-100 (likelihood this is a real issue vs false positive)
- **Location**: file, line number, code snippet
- **Explanation**: why it's a problem
- **Fix**: concrete, actionable suggestion

**Severity definitions:**

| Level | Meaning |
|---|---|
| CRITICAL | Must fix. Bugs, data corruption, security issues. |
| MAJOR | Should fix. Design problems, missing error handling, inadequate testing. |
| MINOR | Recommended. Style inconsistencies, suboptimal patterns, documentation gaps. |
| NIT | Optional. Minor preferences, micro-optimizations. |

### Step 5: Synthesize and report

After collecting all findings:

1. **Merge related issues** — combine findings that point to the same root cause.
2. **Filter false positives** — re-check each finding against the actual code.
3. **Prioritize** — CRITICAL first, then MAJOR, MINOR, NIT.
4. **Number findings** sequentially for easy reference.
5. **Present the report** with: severity, confidence, location, explanation, and fix for each finding.

## Mindset

Be direct. Be specific. Every issue missed is a bug that reaches production.

Do not:
- Add empty praise ("Great job overall!")
- Soften criticism ("Maybe consider...")
- Ignore small issues (they accumulate)
- Assume the author knew better

Do:
- Question everything
- Demand evidence and justification
- Provide concrete alternatives
- Hold the code to the highest standard
