---
name: flutter-testing
description: Use when writing or fixing tests in this repo — unit tests for repositories/use cases/view models, widget tests for a screen's four states, goldens, or integration tests. Includes the fakes pattern, what must be tested, and the mistakes that make tests silently useless.
---

# Flutter testing in this repo

Rule 9 says "design for testing"; this skill is how the tests themselves are written.
Read `test/**` instructions too: `.github/instructions/tests.instructions.md`.

## Index

- [`reference/patterns.md`](reference/patterns.md) — copy-pasteable patterns: fake repository,
  view-model test, four-state widget test, golden test, timeout/401 tests, `textScaler` layout
  test.

## Non-negotiables

1. **No network, no real I/O, no real clocks in a unit or widget test.** Inject fakes; freeze time.
2. **Every async screen is tested in all four states** — loading, error, empty, data — plus retry
   from the error state.
3. **Error paths are tested, not just the happy path**: timeout, 401 (session expired), 403
   (not allowed), 404, malformed body, no connectivity.
4. **A pre-existing bug fixed in this change gets its failing test first**, then the fix.
5. **Tests are deterministic.** No `Future.delayed` races, no `pumpAndSettle` when an animation or
   timer never settles, no dependence on test execution order.
6. **No real tokens, keys, phone numbers or personal data in fixtures** — obviously fake values.

## The fakes-over-mocks rule

Prefer a hand-written fake in `test/fakes/` with fixed fixtures in `test/fixtures/`. It is
readable, reusable, and does not encode call order. Use `mocktail` only when you must verify an
interaction (an event was emitted once, a token refresh happened) — and then use `verifyNever`
for the thing that must **not** happen; that is where the interesting bugs live.

## What a good test name looks like

```
fetches call history and maps it to domain models
returns a typed timeout failure when the client exceeds ten seconds
shows the empty state when there are no calls
wipes the session and routes to login when the refresh token is rejected
keeps the layout valid at 320 dp with textScaler 2.0
```

If a name needs "and" twice, split the test.
