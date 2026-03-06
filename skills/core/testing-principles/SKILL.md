---
name: testing-principles
description: >
  Test type selection (unit/integration/component/E2E), mocking vs faking decisions,
  managed vs unmanaged dependencies. Use when writing tests, deciding what type of test
  to write, whether to mock or fake a dependency, or whether code should be tested at all.
---

# Testing Principles

Based on Vladimir Khorikov, "Unit Testing Principles, Practices, and Patterns".

## Test Type Decision Tree

Classify by **what code is under test**, not by what test doubles are used:

1. **Domain logic or algorithm** (no collaborators that talk to external systems)?
   Unit test. No fakes needed. If you feel the need to fake something, refactor the code.

2. **Adapter code** (code whose job is to talk to an external system — DB, HTTP API, S3, filesystem, message queue)?
   Integration test. **Always.** Use the real external system (Testcontainers, LocalStack, tmp_path).
   Never mock/patch the external system the adapter talks to — that removes the only reason the test exists.

3. **Orchestrating multiple components?**
   Component test. Black-box the service via its API. Real managed deps, fake unmanaged deps.

4. **User journey spanning UI to database?**
   E2E test. Only for the most important business journeys.

5. **None of the above?**
   Probably trivial code. Don't test.

If you feel the need to use a mocking framework: stop. Either the code has no infra dependencies (no mock needed) or it
does (use integration/component test with real or faked deps via DI, not a mocking framework).

### Common mistake: mocking away the adapter's external system

If adapter code contains complex parsing/formatting logic that you want to unit-test, **extract that logic into a pure
function in the domain layer** and unit-test the function. The adapter itself remains thin and is tested only as an
integration test with the real external system. Never create a "unit test" for an adapter by mocking the system it
adapts — the test proves nothing because the only risk is in the real wiring.

## What to Test (Complexity × Collaborators)

|                     | Few Collaborators                     | Many Collaborators                                 |
|---------------------|---------------------------------------|----------------------------------------------------|
| **High Complexity** | Domain model, algorithms → Unit tests | Overcomplicated → Refactor first                   |
| **Low Complexity**  | Trivial → Don't test                  | Controllers/Adapters → Integration/Component tests |

Don't test trivial code (getters, pass-throughs, DTOs). If code is both complex and has many collaborators, split it
into domain logic and orchestration.

## Test Types

- **Unit Tests** — Pure domain logic and algorithms. No infrastructure.
  In hexagonal architecture: the core (validation, calculations, domain rules, data transformations).

- **Integration Tests** — Single adapter against real external system (e.g. Testcontainers).
  One adapter per test, everything else faked. Only verifies wiring — not domain logic.

- **Component Tests** — Entire service from outside via its API. Black box.
  Real managed deps, faked unmanaged deps.

- **E2E Tests** — Critical user journeys from UI to database. Few tests, high value.
  Set up data via API, not UI.

**Frontend** — same four types apply:

- A single UI component rendered against the real DOM with faked API calls is an **integration test**
  (the component is the adapter, the DOM is the external system).
- Test user-visible behavior (click, type, read), never implementation details (state, props, lifecycle).
- Page components → component tests or E2E, not integration tests.

## Managed vs Unmanaged Dependencies

- **Managed** — owned exclusively by the application, not observable from outside.
  Run **real** in tests. Examples: your DB, your storage bucket, your message queue.

- **Unmanaged** — shared with or observable by other systems.
  **Fake** them. Examples: third-party APIs, external auth providers, LLMs.

Mocks and fakes only for unmanaged dependencies. Mocking a managed dep (e.g. in-memory DB fake) hides real behavior. If
tests are slow because of managed deps → better tooling (Testcontainers), not mocking.

Prefer fakes over mocking frameworks. Fakes implement the port interface via DI, break at compile time when interface
changes, and are reusable. Mocking frameworks (`unittest.mock`, `pytest-mock`, `jest.fn()`) only acceptable for
unmanaged deps where a full fake is disproportionate.
