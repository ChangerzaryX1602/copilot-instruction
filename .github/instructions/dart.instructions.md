---
applyTo: "**/*.dart"
---

# Dart rules (naming, null safety, lifecycle, docs, formatting)

Covers rules **3, 6, 7, 10, 15** of the house standard. These are enforced by the analyzer
wherever a lint exists (see `docs/rule-to-lint.md`), so do not argue with them — fix the code.

## Summary

One-line review if you are in a hurry: **no `!`, no empty `catch`, nothing left undisposed,
`snake_case` files only, `lowerCamelCase` identifiers, `///` on public API, trailing commas
on every multi-line call.**

## Naming (rule 6) — Dart's real standard, not a guess

| Thing | Case | Example |
|-------|------|---------|
| File / directory | `snake_case` | `call_history_view.dart`, `use_cases/` |
| Library / `part of` name | `snake_case` | `library call_history;` |
| Package name | `snake_case` | `voice_network` |
| Variable, parameter, function, method, field | `lowerCamelCase` | `callDuration`, `fetchCalls()` |
| **Constant** (`const`, `static const`, enum value) | `lowerCamelCase` | `defaultTimeout`, `CallState.connecting` |
| Class, enum type, typedef, extension, mixin | `UpperCamelCase` | `CallHistoryRepository`, `CallState` |
| Private | leading `_` on the member only | `_controller`, `_fetchInternal()` |

- **Constants are NOT `SCREAMING_CAPS`.** `const defaultTimeout = Duration(seconds: 10);` —
  `DEFAULT_TIMEOUT` is wrong in Dart and the linter flags it (`constant_identifier_names`).
- Acronyms count as one word: `HttpClient`, `parseHtml`, `userId` — not `HTTPClient`, `parseHTML`.
- Do not encode the type in the name (`userNameString`, `listOfCalls`); do not encode the
  framework either (`iUserRepository`, `IUserService` — Dart has no `I`-prefix convention).
- Booleans read as assertions: `isLoading`, `hasError`, `canRetry`, `shouldRefresh`.
- Name things by what they mean in the domain, not by how they are built (`activeCall`, not
  `callModel2`).
- Local variables get no leading underscore (`no_leading_underscores_for_local_identifiers`).

## Null safety and error handling (rule 3)

- **No `!` on anything that can legitimately be null.** There is no official lint for this, so
  it is on you and on the reviewer — write it correctly the first time.

```dart
// ❌ forces an unwrap; crashes in production the moment the API changes shape
final name = response.user!.profile!.displayName!;

// ✅ patterns / switch expressions make absence an explicit, handled case
final name = switch (response.user) {
  final User(:final profile?) => profile.displayName,
  _ => l10n.unknownUser,
};
```

`!` is acceptable only when the invariant is *locally* provable and you say why — e.g. right
after a null check that returns, inside an `if (x != null)` block, or for `late final` fields
assigned in `initState()`:

```dart
if (_controller == null) return const SizedBox.shrink();
final controller = _controller!; // provable: guarded one line above
```

- Prefer **sealed classes + exhaustive `switch`** for UI state so a forgotten state fails to
  compile instead of failing at 2 a.m.:

```dart
sealed class CallState {
  const CallState();
}

final class Idle extends CallState { const Idle(); }
final class Connecting extends CallState { const Connecting(); }
final class Active extends CallState {
  const Active({required this.peer});
  final String peer;
}
final class Failed extends CallState {
  const Failed(this.failure);
  final Failure failure;
}
```

- **Catch typed, at the boundary, and convert.** Network / DB / platform calls are wrapped in
  `try` / `on SomeException catch` in the repository or service — never in a widget:

```dart
/// Loads call history, mapping transport errors to typed failures.
///
/// Returns a [Result] — never throws for expected network conditions.
Future<Result<List<Call>>> fetchCalls() async {
  try {
    final response = await _client.get<List<dynamic>>('/calls').timeout(_timeout);
    return Result.ok(_parse(response.data));
  } on TimeoutException catch (error, stackTrace) {
    _logger.warning('fetchCalls timed out', error: error, stackTrace: stackTrace);
    return const Result.failure(Failure.timeout());
  } on ApiException catch (error, stackTrace) {
    _logger.warning('fetchCalls failed', error: error, stackTrace: stackTrace);
    return Result.failure(Failure.fromStatus(error.statusCode));
  }
}
```

- Never `catch (e) {}`, never `catch (e) { return null; }`, never swallow and continue — a
  silent catch hides an outage. If you truly must ignore, catch a *specific* type and put a
  comment on the same line explaining why it is safe.
- Do not catch `Error` (`StateError`, `TypeError`, …) — those are programming bugs; let them
  crash in development and be reported in production.
- Throw only `Exception` subtypes (`only_throw_errors`). Never `throw 'something went wrong'`.
- Never `throw` inside `finally` (`throw_in_finally`) and never `return` from `finally`.
- Every `Future` that can throw is either `await`ed inside a `try`, or explicitly handled:
  `unawaited(future.catchError(...))` with a comment. Fire-and-forget without a handler is a
  finding (`unawaited_futures`, `discarded_futures`).
- Do not compare values of unrelated types (`unrelated_type_equality_checks`), and never use
  `dynamic` for new code — `avoid_dynamic_calls` keeps the door shut.
- `buildContext` is never used after an `await` without a mounted/`context.mounted` guard
  (`use_build_context_synchronously`).

## Lifecycle and memory (rule 7)

Everything you create, you destroy — and `super.dispose()` is the **last** statement.

```dart
class CallScreen extends StatefulWidget {
  const CallScreen({required this.viewModel, super.key});

  final CallViewModel viewModel;

  @override
  State<CallScreen> createState() => _CallScreenState();
}

class _CallScreenState extends State<CallScreen> {
  late final TextEditingController _searchController;
  late final FocusNode _searchFocus;
  StreamSubscription<CallEvent>? _events;

  @override
  void initState() {
    super.initState();
    _searchController = TextEditingController();
    _searchFocus = FocusNode();
    _events = widget.viewModel.events.listen(_onEvent);
  }

  @override
  void dispose() {
    _events?.cancel();          // 1. subscriptions and timers first
    _searchFocus.dispose();     // 2. reverse order of creation
    _searchController.dispose();
    super.dispose();            // 3. always last
  }
}
```

- Also dispose: `AnimationController`, `ScrollController`, `PageController`,
  `TabController`, `TransformationController`, `ValueNotifier`, `ChangeNotifier` you own,
  `Timer`, `RestorableProperty` (`dispose()` from `RestorationMixin`).
- `addListener` / `addObserver` must have a matching `removeListener` /
  `removeObserver` (`WidgetsBindingObserver`, `ScrollController`, streams).
- Do not create controllers inside `build()`; create them in `initState` or as `late final`
  fields.
- `setState` after `dispose` is a crash: guard with `if (!mounted) return;`.
- `StreamController`, `Sink`, `RawDatagramSocket`, `HttpClient`: `close()` / `cancel()`
  (`close_sinks`). An unbounded `StreamController` in a long-lived screen is a leak.
- Memory-safe habits the linter checks: `avoid_slow_async_io` (use the `dart:io` async APIs),
  `cancel_subscriptions`, `close_sinks`.

## Documentation (rule 10)

- `///` doc comments on every **public** API: class, method, function, field, typedef.
  `//` (two slashes) is for implementation notes only (`slash_for_doc_comments`).
- Shape: **one-line summary ending with a period**, then blank line, then parameters /
  return / throws when they are not obvious from the name.

```dart
/// Fetches call history for the signed-in subscriber.
///
/// [page] is 1-based. Returns an empty page — not an error — when the subscriber
/// has no calls. Throws [AuthFailure] when the session is no longer valid.
Future<Result<CallPage>> fetchCalls({required int page}) async { … }
```

- Refer to code with brackets so the reference is checkable: `[CallPage]`, `[page]`
  (`comment_references`).
- Write **why**, not what. `/// Increments i by one.` adds nothing; `/// Retries once because
  the PBX drops the first INVITE after an idle period.` is worth its bytes.
- Keep it concise — no changelog, no author tags, no spec restating the signature. And no
  lies: if the doc says "returns empty when…", the code must do that.
- TODOs carry an owner: `// TODO(chang): remove after backend v2 ships`
  (`flutter_style_todos`).

## Formatting (rule 15)

- **Trailing comma on every multi-line argument list, parameter list, list/map literal.**
  It is what tells `dart format` to break the layout vertically and keeps diffs one-line-per-
  argument (`require_trailing_commas` enforces it).

```dart
Widget build(BuildContext context) {
  return CallTile(
    peer: call.peer,
    duration: call.duration, // ← trailing comma forces the vertical layout below too
  );
}
```

- `dart format .` after every edit; CI runs `dart format --output=none --set-exit-if-changed`.
  Never hand-format to beat the formatter.
- `page_width: 100` and `trailing_commas: preserve` come from `analysis_options.yaml`; do not
  override per file. A longer line is fine when the formatter produced it — do not wrap by hand.
- Single quotes by default (`prefer_single_quotes`); double quotes only when the string
  contains a single quote (`'it\'s'` → `"it's"`).
- Imports: `dart:` first, then `package:` (alphabetical), then relative — actually use
  `package:` imports for anything inside `lib/` (`always_use_package_imports`,
  `directives_ordering`). Never import across features by relative path.
- Do not use `print` anywhere in `lib/` (`avoid_print`) — use the logging layer so the log can
  be redacted and disabled in release.
- Async functions return `Future<T>`; only use `void` for genuine fire-and-forget
  (`avoid_void_async`).
