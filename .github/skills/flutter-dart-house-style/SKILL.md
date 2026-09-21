---
name: flutter-dart-house-style
description: Use when writing, refactoring or reviewing Flutter/Dart code in this repo and you need the full house standard — the 15 rules with why/good/bad examples, SOLID in Dart terms, docs style and formatting. Load for any non-trivial Dart change, refactor or code review.
---

# Flutter/Dart house style

The path-scoped instruction files are the short version. **This skill is the long version** —
load it when a change is non-trivial, when a rule and convenience disagree, or when reviewing.

## Index — start here

- [`reference/the-15-rules.md`](reference/the-15-rules.md) — each rule: what, why it matters,
  good/bad code, what enforces it, and how it fails in production.
- [`docs/rule-to-lint.md`](../../../docs/rule-to-lint.md) — which rule is machine-enforced and
  which is convention only (read this before promising someone a rule is "checked by CI").
- Rule-by-rule one-liners: `.github/instructions/dart.instructions.md` (3,6,7,10,15),
  `flutter-ui.instructions.md` (4,5,11,12,13), `architecture.instructions.md` (1),
  `dependencies.instructions.md` (2,14), `security.instructions.md` (14), `tests.instructions.md` (9).

## The five mistakes to check first

These are the ones that actually happen — check them before anything else:

1. `!` used to make the compiler stop complaining, on a value that can be null at runtime.
2. A network/repository call inside `build()` or `initState` of a widget.
3. A controller, subscription, timer or notifier created and never disposed.
4. `catch (e) {}` — an empty or a type-less catch that swallows an outage.
5. A `_buildX()` method returning a Widget instead of a `const`-able `StatelessWidget` class.

## SOLID in Dart, concretely

- **S** — one reason to change: a view model renders state; it does not also talk to the API.
- **O** — extend by adding a case (sealed class + exhaustive `switch`) or a new implementation
  of the interface, not by adding `if (type == 'x')` branches to existing code.
- **L** — a subclass that throws `UnimplementedError` (or `throw UnimplementedError()` on an
  inherited method) breaks substitutability. Split the interface instead.
- **I** — many small interfaces beat one fat one. A repository for calls should not expose
  methods only the settings screen needs.
- **D** — depend on the `domain/` abstraction, not the concrete client. This is also what makes
  rule 9 (testability) possible.

## DRY — the rule of three

Do **not** extract on the second occurrence. Two similar blocks are a coincidence; the third is
a pattern. Extract when:

- the third copy appears, or
- the logic is a domain rule (it must live in `domain/` even if used once), or
- a bug was fixed in one copy and the other copies still have it.

Over-abstraction (a "generic" widget with 9 optional parameters used once) is a worse defect
than duplication.

## Documentation style this repo wants

```dart
/// Cancels the active call and returns once the SIP session is confirmed closed.
///
/// Throws [CallAlreadyEndedException] if the call ended while the request was in flight —
/// callers should treat that as success, not as an error to surface.
Future<void> cancelCall(String callId) async { … }
```

One-line summary ending in a period · blank line · parameters and non-obvious exceptions ·
why, never a restatement of the code. `//` for implementation notes. `// TODO(name): …`.

## Formatting and naming gotchas

- Trail**ing** commas are not cosmetic: they are how `dart format` knows to lay a call out
  vertically. Add them and let the formatter decide the line breaks.
- Dart constants are `lowerCamelCase` (`defaultTimeout`), **never** `SCREAMING_CAPS`.
- Only file and directory names use `snake_case`.
- No `print` in `lib/` — the logging layer exists so output can be redacted and turned off.
