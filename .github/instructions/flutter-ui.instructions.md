---
applyTo: "lib/**/*.dart"
---

# Flutter UI rules (rules 4, 5, 11, 12, 13)

Covers: `const` + widget extraction, logic-vs-UI separation, one state-management pattern,
no hardcoded strings, responsive/adaptive UI — plus accessibility, which the original list
implied but never stated.

## Rule 5 — Views render. ViewModels decide.

A widget's job: take state in, render it, forward intent out. Nothing else.

**Forbidden inside a `Widget` / `State`:** `http`/`dio` calls, repository calls, JSON parsing,
business rules (pricing, eligibility, formatting rules), `SharedPreferences` access, timers that
change app state, navigation decisions based on domain rules.

```dart
// ❌ the widget owns the request, the parse and the error policy
class _CallListState extends State<CallList> {
  List<Call>? calls;
  @override
  void initState() {
    super.initState();
    Dio().get('/calls').then((r) => setState(() => calls = r.data as List<Call>));
  }
  ...
}

// ✅ the widget renders a state snapshot and sends intents
class CallListView extends StatelessWidget {
  const CallListView({required this.viewModel, super.key});

  final CallListViewModel viewModel;

  @override
  Widget build(BuildContext context) {
    return switch (viewModel.state) {
      CallListLoading() => const CallListSkeleton(),
      CallListFailed(:final failure) => ErrorView(
          failure: failure,
          onRetry: viewModel.load,
        ),
      CallListEmpty() => EmptyView(message: context.l10n.noCalls),
      CallListData(:final calls) => CallListBody(calls: calls),
    };
  }
}
```

- Formatting for display (dates, durations, currency) lives in a formatter in `core/`, called
  by the view or the view model — not inline with `if`/`switch` domain rules.
- Navigation: the view may navigate *because an intent succeeded*; it must not decide *whether*
  an action is allowed.
- `build()` is pure: no I/O, no logging side effects, no `setState`, no mutations. It may run
  60×/s and must survive that.

## Rule 11 — One state-management pattern, chosen once

- Use the pattern pinned in `.github/copilot-instructions.md` (Riverpod / Bloc / provider /
  `ChangeNotifier` + `ValueNotifier`) for **all** app state.
- **Never mix** in the same screen or feature: no `setState` driving business state next to a
  Bloc, no `Provider.of` reading a Bloc, no half-migrated feature.
- `setState` is allowed **only** for purely local, ephemeral UI state (a `TabBar` index, a
  checkbox in a dialog, an animation toggle) that nobody else observes.
- State exposed to the UI is immutable. Mutable state stays inside the notifier/bloc; the
  outside sees a snapshot (`copyWith` / sealed state classes / `AsyncValue`).
- One screen = one view model/controller. A 500-line controller with 12 concerns gets split by
  concern, not by screen.
- Dispose/close every notifier and stream the view model owns.

## Rule 4 — `const` and widget extraction

- **`const` wherever the analyzer allows.** `const` constructors, `const` literals, `const`
  fields; a `const` widget subtree is not rebuilt — that is free performance.
  (`prefer_const_constructors`, `prefer_const_constructors_in_immutables`,
  `prefer_const_literals_to_create_immutables`, `prefer_const_declarations`.)
- Mark the widget constructor `const` and take `super.key`.

```dart
// ✅ stateless, const-constructible, testable in isolation
class CallTile extends StatelessWidget {
  const CallTile({required this.peer, required this.duration, super.key});

  final String peer;
  final Duration duration;

  @override
  Widget build(BuildContext context) => ListTile(
        title: Text(peer),
        subtitle: Text(context.l10n.durationLabel(duration)),
      );
}
```

- **Extract whole widgets, not build methods.** A `Widget _buildHeader()` method or a
  `Widget header(BuildContext c)` function is an anti-pattern: it cannot be `const`, it rebuilds
  with the parent, it hides in the tree, and it cannot be widget-tested. Make a class.

```dart
// ❌ private method: no const, no isolation, rebuilds with the parent
Widget _buildHeader() => Text(context.l10n.title);

// ✅ separate const-able widget
class _Header extends StatelessWidget {
  const _Header();
  @override
  Widget build(BuildContext context) => Text(context.l10n.title);
}
```

- Keep `build()` readable: nesting ≤ 4 levels, no more than ~50 lines; beyond that, extract.
- Lists are lazy: `ListView.builder` / `SliverList.builder` / `GridView.builder` —
  never `Column(children: items.map(...).toList())` for dynamic content.
- `MediaQuery` reads in hot widgets use the narrow accessor: `MediaQuery.sizeOf(context)`,
  `MediaQuery.paddingOf(context)`, `MediaQuery.textScalerOf(context)` — not `.of(context)`.
- Avoid layout thrash: no `Opacity` inside rebuild-heavy trees (`AnimatedOpacity` /
  `FadeTransition`), `RepaintBoundary` around independently animating subtrees, `const` above
  them.
- Images: `cacheWidth`/`cacheHeight` for thumbnails, `gaplessPlayback` to stop flicker, and a
  placeholder + error widget. Full-resolution images in a list are a memory bug on Android.
- `const` keys / stable keys for list items that reorder (`ValueKey(id)`), never the index for
  a mutable list.

## Rule 12 — No hardcoded strings

- No user-visible literal in a widget: text, hint, label, tooltip, snackbar, dialog button,
  error message, semantic label, date/number pattern.
- All copy comes from the l10n layer (`lib/l10n/`, generated from `*.arb`). Access is
  `context.l10n.<key>` (or the app's equivalent); a plain `const` strings class is acceptable
  only for a single-language app that will never ship a second language — and it must still be
  centralised, one file, no exceptions.
- **No string concatenation to build a sentence.** Use a parameterised message with
  placeholders so word order is translatable: `l10n.greeting(name)` — not `'สวัสดี ' + name`.
- Plural / gender / date / number formatting goes through the l10n layer (ICU syntax), never
  `'$count รายการ'`.
- Also localise: error messages the user sees, push-notification copy, and `Semantics` labels.
- A language is not a feature flag: adding a locale means adding an `.arb`, not `if (locale ==
  'th')` in a widget.
- Log messages, exception messages and API paths are **not** user-visible copy — keep those in
  code (English), but never include PII (see `security.instructions.md`).

## Rule 13 — Responsive and adaptive

- No fixed `width`/`height` for layout. Allowed fixed sizes: icons, avatars, dividers,
  `AspectRatio` boxes, stroke widths — things whose size genuinely is absolute.
- Prefer, in order: `Expanded` / `Flexible` / `Spacer` → constraints
  (`ConstrainedBox`, `SizedBox.shrink`) → `LayoutBuilder` → `MediaQuery.sizeOf` directly.
- Breakpoints come from **one** place (`core/theme/breakpoints.dart`), not magic numbers
  scattered per screen.
- Handle the real cases, not just "phone portrait": narrow (320 dp), typical phone, tablet /
  unfolded (> 600 dp), landscape, split-screen, and the keyboard being open.
- Scrollable content: `SingleChildScrollView` / `CustomScrollView` + `SafeArea`; a form must
  still work when the keyboard covers half the screen (`resizeToAvoidBottomInset`,
  `scrollPadding`).
- Text scaling is a requirement, not an edge case: layouts must survive
  `textScaler` up to 2.0 without overflow (`TextOverflow.ellipsis`, `maxLines`, `Flexible`
  around long labels). Overflow errors are defects, not warnings.
- Adaptive, not only responsive: platform-appropriate behaviour via `Theme.of(context)` /
  platform checks in `core/`, and `TargetPlatform` differences (back gesture, dialogs,
  date pickers) rather than hardcoding Cupertino widgets everywhere.
- Never assume orientation, locale, RTL, or a notch. Use `SafeArea` / `MediaQuery.paddingOf`.
  RTL-safe layouts use `start`/`end` (EdgeInsetsDirectional, AlignmentDirectional,
  `TextAlign.start`), never `left`/`right`, when the app may ship an RTL language.
- Verify with a widget test at two sizes (at least 320 dp and ~800 dp) for each screen that
  claims to be responsive.

## Accessibility (not in the original list — still not optional)

- Touch targets ≥ 48×48 dp: a small icon inside an `IconButton` with `constraints`/padding.
- `Semantics` / `semanticLabel` on icon-only buttons and images; no image-button without a
  label.
- Contrast: use theme colours, not custom greys (`Theme.of(context).colorScheme`); test in
  dark mode.
- Never convey information by colour alone — pair with text/icon.
- Screen-reader order follows reading order: avoid `Stack` where a `Column` works, and set
  `sortKey` when order must differ from paint order.
- Respect `disableAnimations` / `MediaQuery.disableAnimationsOf(context)` and keep essential
  functionality reachable without motion.

## Theming and visual consistency

- Colours, spacing, radii and text styles come from tokens in `core/theme/` — no inline
  `Color(0xFF…)`, no magic `EdgeInsets.all(13)`.
- Widgets read `Theme.of(context)` / `TextTheme`; a screen that fights the theme is a defect.
- Loading, error and empty states are designed, not defaults: skeleton (preferred) or
  `CircularProgressIndicator`, an error view with a retry action, an empty view with a hint —
  **all four states on every async screen.**
