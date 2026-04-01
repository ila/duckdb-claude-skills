---
name: best-practices
description: Generic development rules for DuckDB extension projects. Auto-loaded when writing code, reviewing changes, or discussing development practices. Covers testing, code style, git safety, debugging, and collaboration.
---

# DuckDB Extension Development Best Practices

## Working with the user

When you're stuck — either unable to fix a bug after 2-3 attempts, or tempted to work around the actual problem by redefining the objective — **stop and ask the user for directions**. Explain clearly what the specific problem is. The user knows this codebase deeply and can often point you to the right solution in one sentence. Do not silently change the goal, declare something impossible, or add bloated workarounds without consulting first. We work as a team.

Always test your changes with real queries before declaring success, not just unit tests. Unit tests with wide thresholds can pass even when the code is fundamentally broken.

Never execute git commands that could lose code. Always ask the user for permission on those.

## Development rules

- **New features must have tests.** Ask the user whether to create a new test file or extend an existing one in `test/sql/`.
- **Never remove a failing test to "fix" a failure.** If a test fails, fix the underlying bug. Tests exist for a reason.
- **Never weaken tests to match current behavior.** If a test exposes a bug, the test stays and the bug gets fixed. Never limit test scope to avoid known failures, skip broken scenarios, mark tests as expected-fail, comment them out, or convert them to TODOs. Write the test for the correct behavior, then make the code pass it.
- **Before implementing anything, search the existing codebase** for similar patterns or solutions. Check if a helper function, utility, or prior approach already addresses the problem. Reuse before reinventing.
- **Use helper functions.** Factor shared logic into helpers rather than duplicating code.
- **Never edit the `duckdb/` submodule.** The DuckDB source is read-only. All extension logic lives in `src/` and `test/`. If you need DuckDB internals, use the public API or ask the user.

## Build & test

```bash
GEN=ninja make                # build (release)
GEN=ninja make debug          # build (debug)
make test                     # run all tests
make format-fix               # auto-format code
```

Run a single test:

```bash
build/release/test/unittest "test/sql/<test_name>.test"
```

## Code style (clang-tidy)

DuckDB extensions follow DuckDB's clang-tidy configuration. Key naming rules:

- **Classes/Enums**: `CamelCase`
- **Functions**: `CamelCase`
- **Variables/parameters/members**: `lower_case`
- **Constants/static/constexpr**: `UPPER_CASE`
- **Macros**: `UPPER_CASE`

Other style rules (from `.clang-format`, based on LLVM):

- **Tabs for indentation**, width 4
- **Column limit**: 120
- **Braces**: same line as statement (K&R / Allman-attached)
- **Pointers**: right-aligned (`int *ptr`, not `int* ptr`)
- **No short functions on single line**
- **Templates**: always break after `template<...>`
- **Long arguments**: align after open bracket

Run `make format-fix` to auto-format.

## Debugging

Use a project-specific debug print macro (e.g., `MY_EXT_DEBUG_PRINT`) that compiles out when disabled. Add prints at major code flow points: entry/exit of compilation phases, rewrite rules, and key decisions.

Use `EXPLAIN` to see transformed query plans.

## Git safety

- Never run destructive git commands without asking the user
- Prefer new commits over amending
- Never skip hooks (`--no-verify`)
- Never force push to main
