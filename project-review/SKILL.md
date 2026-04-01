---
name: project-review
description: Subsystem-by-subsystem codebase audit. Discovers source directories, reviews each from 5 perspectives, and produces prioritized findings. Use for periodic quality sweeps or before releases.
---

# Project Review

Systematic codebase audit that reviews each subsystem from multiple perspectives and produces actionable findings.

## Process

### Step 1: Discover subsystems

Identify subsystems from the `src/` directory structure. Each immediate subdirectory of `src/` (or `src/include/`) is a subsystem. Also treat `test/sql/` and `docs/` as reviewable subsystems.

```bash
# List source subsystems
ls -d src/*/
# List test and doc areas
ls test/sql/ docs/
```

### Step 2: For each subsystem

Process subsystems one at a time. For each:

#### 2a: Read all source files

Read every `.cpp` and `.hpp` file in the subsystem. Understand:
- What this subsystem does
- How it interacts with other subsystems
- What its public API is

#### 2b: Launch 5 review agents

Same 5 perspectives as [code-review](../code-review/SKILL.md), but applied to the entire subsystem (not just a diff):

1. **Correctness** — Are algorithms correct? Edge cases handled? Invariants maintained?
2. **Reliability** — Is there test coverage? Error handling? Crash-safe state management?
3. **Performance** — Hot paths optimized? Unnecessary allocations? Algorithmic complexity?
4. **Simplicity** — Dead code? Over-abstractions? Could anything be simplified?
5. **Style** — Naming conventions? Debug prints? Documentation completeness?

Provide each agent with the full source code of the subsystem plus relevant context from CLAUDE.md and neighboring subsystems.

#### 2c: Collect findings

Each agent produces findings with:
- **Severity**: CRITICAL / MAJOR / MINOR / NIT
- **Confidence**: 0-100
- **Location**: file:line
- **Explanation**: why it's a problem
- **Fix**: concrete suggestion

### Step 3: Synthesize per subsystem

For each subsystem:
1. Merge related findings across perspectives
2. Filter false positives (re-read the code to verify)
3. Number findings sequentially
4. Present prioritized report

### Step 4: Track progress

After completing each subsystem, report:
- Subsystems completed / total
- Findings by severity (CRITICAL: N, MAJOR: N, MINOR: N, NIT: N)
- Next subsystem to review

### Step 5: Create action items

For all CRITICAL and MAJOR findings, create TODO items describing:
- The problem
- The affected file and line
- The suggested fix
- Priority (CRITICAL fixes first)

## Severity Definitions

| Level | Meaning | Action |
|---|---|---|
| CRITICAL | Bugs, data corruption, security issues | Must fix before release |
| MAJOR | Design problems, missing error handling, inadequate testing | Should fix soon |
| MINOR | Style inconsistencies, suboptimal patterns, doc gaps | Fix when touching the file |
| NIT | Minor preferences, micro-optimizations | Optional |

## When to Use

- Before a release or paper submission
- After a major refactor
- When onboarding to understand codebase quality
- Periodic quality sweeps (monthly/quarterly)
