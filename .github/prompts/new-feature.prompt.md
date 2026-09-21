---
mode: agent
description: "เพิ่ม feature ใหม่แบบเดินครบทุก layer ตามกฎข้อ 1"
---

# Add a feature (layer walk)

Add the feature below to this Flutter app. Follow the layering in
`.github/instructions/architecture.instructions.md` — **in this order**, one step at a time,
showing the code for each step before moving on.

Feature: `<describe the feature, the screen(s) involved, and what the user sees>`

## Step 1 — domain (pure Dart, no Flutter, no JSON)

- `lib/features/<feature>/domain/entities/<entity>.dart` — immutable entity with `const`
  constructor, `final` fields, `copyWith`, value equality if not using freezed.
- `lib/features/<feature>/domain/repositories/<x>_repository.dart` — abstract interface with
  the smallest set of methods the feature needs, returning `Result<T>` / typed failures.

## Step 2 — data

- `.../data/models/` — DTO with generated `fromJson`/`toJson`. Missing/null fields map to a
  defined default or a typed failure; no `as` casts.
- `.../data/datasources/<x>_remote_datasource.dart` — one method per endpoint, timeout on every
  call, `on SpecificException catch` → typed failure. No parsing, no caching here.
- `.../data/repositories/<x>_repository_impl.dart` — implements the domain interface: DTO →
  entity mapping, cache policy, bounded retry, error mapping. This is the only place that knows
  both DTOs and entities.

## Step 3 — presentation

- `.../presentation/view_models/<x>_view_model.dart` — immutable state, the four states
  (loading / error / empty / data), intents as methods, dependencies via constructor, `dispose()`
  for anything it owns.
- `.../presentation/views/<x>_view.dart` — `StatelessWidget` (or `StatefulWidget` only for
  local UI state), renders `switch (viewModel.state)` exhaustively, no I/O, no business rules,
  no hardcoded strings.
- `.../presentation/widgets/` — extracted `const`-able widgets. No `_buildX()` methods.

## Step 4 — wiring and routing

- Register the new repository/datasource/view model in the composition root only.
- Add the route in `core/router/` if this is a new screen; typed route, ids not objects.

## Step 5 — l10n, a11y, responsive

- All copy through the l10n layer (add the ARB entries).
- Semantic labels on icon-only controls; touch targets ≥ 48 dp.
- Layout survives 320 dp, tablet width, landscape, and `textScaler` 2.0.

## Step 6 — tests and verification

- Unit: repository mapping + every error path (timeout, 401, 403, malformed body).
- Widget: the four states, plus retry from the error state.
- Then run: `dart format .` · `flutter analyze --fatal-infos` · `flutter test`.

Finish by reporting: files added/changed, the exact commands you ran, their real result, and
anything you could not verify. If a step requires a package that is not on the allowlist, stop
and ask before adding it.
