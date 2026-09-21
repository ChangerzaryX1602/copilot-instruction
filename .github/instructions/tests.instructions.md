---
applyTo: "test/**,integration_test/**,**/*_test.dart,**/test_driver/**"
---

# Tests (rule 9 — design for testability, plus what must actually be tested)

## Testable by construction (rule 9)

- **Constructor injection only.** A class that creates its own dependency
  (`final _repo = UserRepository()`, `Dio()` inside a method, a locator lookup inside a widget)
  cannot be tested — that is a design defect, not a testing problem.
- No global mutable state, no `static` caches, no `DateTime.now()` / `Uuid()` / `Random()`
  called directly in logic — inject `Clock`, `Uuid`, `Random`, `Connectivity`. Tests must be
  deterministic and time-independent.
- Depend on the **abstract** repository/interface (defined in `domain/`), not the
  implementation, so the test can substitute a fake.
- Keep platform channels behind an interface in `data/services/` and fake them in tests.
- Prefer plain fakes over mocks where practical; use `mocktail` when verifying interactions.
  Never mock what you can construct.
- Widgets take their data as constructor parameters (state in, callbacks out) so a widget test
  needs no DI container.

## The pyramid this repo expects

| Level | Scope | Tool | Speed |
|-------|-------|------|-------|
| Unit | entities, use cases, repositories (mapping, cache, retry, error mapping), formatters, validators | `test` + `mocktail` | ms |
| Widget | one screen or widget against a fake repository, **all four states** | `flutter_test` | 10s |
| Golden | shared/design-system widgets and the app shell (`matchesGoldenFile`) | `flutter_test` | 10s |
| Integration | real user journeys against a fake or staging server | `integration_test` | minutes |

Coverage target: **new/changed code ≥ 80%**, and the CI gate enforces a floor for the package —
do not lower the floor to make a PR pass. Coverage is a floor, not a goal: a test that asserts
nothing is worse than no test.

## What must be tested (not optional)

- The **error paths**: 400/401/403/404/500, timeout, no connectivity, malformed JSON.
- Every **four-state screen**: loading, error, empty, data — plus retry from error, and the
  empty state copy.
- **Ownership/permission cases** for anything personal: a user must not see another user's data
  when the API is buggy (the client must not "fall back" to showing something).
- **Boundary values**: empty list, one item, very long text, huge numbers, unicode/Thai text,
  RTL text, `textScaler` 2.0, 320 dp and tablet width.
- **Mapping logic**: DTO → entity, especially null/missing fields — a missing field must produce
  a defined default or a typed failure, never a crash.
- Anything you fixed as a bug: add the failing test first, then the fix (a regression without a
  test will come back).

## Conventions

- Test names read as behaviour in plain language:
  `fetches call history and maps it to domain models`,
  `shows the empty state when there are no calls`,
  `wipes the session and routes to login when the refresh token is rejected`.
- Structure: arrange / act / assert with a blank line between; one behaviour per test; no
  logic in the test (`if`, loops over expected values) — that is how tests quietly stop testing.
- Find widgets by `Key`, text, or semantic label — never by private internals, widget type
  indexes, or `find.byType(Container)` chains. Use `ValueKey`s in the UI on purpose, for tests.
- Fake repositories live in `test/fakes/` with fixed fixtures in `test/fixtures/`; do not
  duplicate JSON literals across test files.
- `pumpAndSettle` only when a real animation settles; otherwise `pump(Duration)` — a
  `pumpAndSettle` that times out usually means an infinite animation or a pending timer, which
  is a real bug worth fixing.
- Every test cleans up: dispose controllers/notifiers, close streams, `tearDown` restores
  overrides and fakes. Leaked timers fail the suite for a reason.
- Never hit the real network in a unit or widget test. No real API keys, tokens, or personal
  data in fixtures — use obviously fake values (`0900000000`, `test@example.invalid`).
- Do not test the framework, and do not test generated code. Test your logic, mapping, and
  user-visible behaviour.
- Skipped or commented-out tests are findings: either fix them within the same change or delete
  them and say why in the PR.
