# Coding Standards & Quality Guidelines

> **When to load:** This file should be read when performing **Implementation** or **Refactoring** tasks.

## 1. Codebase Understanding First (CRITICAL)

- **Read Before Write:** NEVER propose code changes without reading the existing code first. Understand the module's style, abstractions, and conventions before editing.
- **Trace Dependencies:** When modifying a function/class, read its callers and callees to understand impact. Use `grep` or semantic search to discover usage patterns.
- **Discover Existing Patterns:** Before creating new utilities or abstractions, search if similar ones already exist. Avoid reinventing the wheel.
- **Understand Domain Context:** Read `OVERVIEW.md` or equivalent docs to grasp the business domain. Code without domain understanding leads to technical correctness but semantic errors.
- **Check Tests:** If tests exist, read them to understand expected behavior and edge cases.

## 2. Architecture & Design Principles

- **SOLID & Design Patterns:** Strictly adhere to SOLID principles. Apply standard design patterns (Factory, Strategy, Singleton, Adapter) *only where appropriate* to solve specific problems, not for complexity's sake.
- **Dependency Injection:** Prefer injecting dependencies (via constructors or interfaces) rather than hard-coding imports/instantiations to ensure testability and loose coupling.
- **Immutability:** Prefer immutable data structures. Avoid side effects in functions unless explicitly required by the operation.

## 3. Defensive Programming & Security

- **Fail Fast & Guard Clauses:** Use "Guard Clauses" at the beginning of functions to handle invalid states immediately. Avoid deep nesting (`if/else` hell).
- **Input Validation:** Never trust input. Validate data boundaries, types, and formats at the entry point of public methods/API handlers.
- **Security First:** Sanitize inputs to prevent Injection attacks (SQLi, XSS). Never commit secrets/API keys (use environment variables).

## 4. Performance & Efficiency

- **Complexity Awareness:** Be mindful of Big O notation. Avoid nested loops ($O(n^2)$) on potentially large datasets. Propose efficient data structures (Sets, Maps) over Arrays for lookups ($O(1)$ vs $O(n)$).
- **Async/Concurrency:** Handle asynchronous operations properly (e.g., proper `await`, `Promise.all` for parallel tasks). Avoid blocking the main thread/event loop.
- **Database Optimization:** Detect and prevent N+1 query problems. Suggest indexing for fields frequently used in filters/joins.

## 5. Code Clarity & Observability

- **Cognitive Load Reduction:** Use strictly typed interfaces/DTOs. Avoid "Magic Numbers" or "Magic Strings" – extract them into named Constants or Enums.
- **Self-Documenting Code:** Variable and function names must reveal intent (e.g., `isUserActive` instead of `flag`).
- **Observability:** When handling errors, ensure logs contain *context* (IDs, input state), not just the stack trace. Code should be debuggable in production.

## 6. Incremental Development & Change Safety

- **Small, Focused Changes:** Make one logical change at a time. Avoid "big bang" commits that touch multiple unrelated concerns.
- **Preserve Existing Behavior:** When refactoring, ensure existing functionality remains intact. Use the same interfaces unless explicitly changing them.
- **Leave Code Better (Slightly):** Apply the Boy Scout Rule—improve only what you touch, don't refactor the entire module when fixing a small bug.
- **Backward Compatibility:** Consider callers when changing function signatures. Prefer adding optional parameters over breaking changes.
- **Validate After Each Step:** Run linters/tests after each significant change, not just at the end.

## 7. Pattern Consistency & Convention Adherence

- **Follow Existing Style:** Match the repo's naming conventions (camelCase vs snake_case), file organization, and import ordering. Consistency > personal preference.
- **Reuse Existing Abstractions:** If the codebase uses a specific pattern (e.g., Repository pattern, Service layer), follow it even if you'd prefer something different.
- **Module Boundaries:** Respect existing package/module boundaries. Don't create cross-cutting dependencies that violate the architecture.
- **Error Handling Convention:** Match how errors are handled elsewhere—exceptions vs Result types, custom error classes vs generic ones.
- **Configuration Pattern:** Use the same approach for configs (env vars, config files, DI containers) as the rest of the codebase.

## 8. Effective Naming & Organization

- **Names Reveal Intent:** Use descriptive names that clearly communicate purpose.
- **Consistent Naming Across Layers:** Use the same terminology as the domain—don't rename concepts between layers.
- **File Organization:** One concept per file. If a file grows beyond 300-400 lines, consider splitting by responsibility.
- **Import Clarity:** Prefer explicit imports over wildcard imports. Group imports: stdlib → third-party → local.
- **Comment the "Why":** Code shows *what*; comments explain *why* – business reasons, non-obvious tradeoffs, or workarounds.

## 9. Error Handling & Edge Cases

- **Handle the Unhappy Path:** Consider: What if the input is empty? What if the API times out? What if the list has 0 or 1 element?
- **Specific Exceptions:** Catch specific exceptions, not bare `except:`. Re-raise with context when appropriate.
- **Fail with Context:** Error messages should include *what* failed, *why* it failed, and *what was the input*.
- **Graceful Degradation:** When possible, provide fallback behavior rather than crashing (e.g., use cached data if API fails).
- **Resource Cleanup:** Ensure resources (files, connections) are properly closed—use context managers (`with` statement) or try/finally.

## 10. Minimalism & Avoiding Over-Engineering

- **Do Only What's Asked:** Resist the urge to add "nice-to-have" features. Solve the current problem, not hypothetical future ones.
- **No Premature Abstraction:** Don't create a generic solution for a one-time problem. Wait until you have 3 similar cases before abstracting.
- **Simplest Solution First:** Start with the straightforward implementation. Optimize only when there's evidence of a performance problem.
- **Avoid Gold Plating:** A working solution today is better than a perfect solution next week. Ship incrementally.
- **Question Complexity:** If your solution requires many new classes/functions, step back and ask if there's a simpler approach.
