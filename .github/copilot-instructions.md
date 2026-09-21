# Repository instructions (Flutter / Dart)

> Always-on rules — read on **every** request. Keep this file under one page.
> Anything longer belongs in `.github/instructions/` (path-scoped) or `.github/skills/` (on demand).
> Fill every `<REPLACE: …>` marker from the real `pubspec.yaml` before committing.
> ไม่มี application code ในชุดนี้ — เป็นกฎเท่านั้น

## 1. Stack — pinned on purpose

The model's training data is older than this repo. **Never emit idioms from another major
version.** If an API was renamed or removed in the pinned version, stop and say so instead of
writing the old call.

- Flutter `<REPLACE: 3.x stable>` · Dart `<REPLACE: 3.x>` (must match `environment: sdk:` in `pubspec.yaml`)
- State management: `<REPLACE: riverpod / bloc / provider / change_notifier — ONE, never mixed>`
- HTTP client: `<REPLACE: dio / http>` · JSON: `<REPLACE: freezed + json_serializable / built_value / manual>`
- Routing: `<REPLACE: go_router / auto_route / Navigator 2.0>`
- Storage: `flutter_secure_storage` (tokens only) + `<REPLACE: shared_preferences / hive / sqflite>`
- Lints: `analysis_options.yaml` (this repo) · Format: `dart format`, `page_width: 100`
- Tests: `flutter_test` + `<REPLACE: mocktail / mockito>` · golden tests: `<REPLACE: yes / no>`
- Minimum OS: Android `<REPLACE: API level>` · iOS `<REPLACE: version>`

## 2. Non-negotiable rules

1. **No secrets in the app, ever.** A Flutter binary is public — it can be unpacked. No API
   secret, signing key, admin token, or DB credential in source, in a `.env` committed to git,
   in logs, or in commit messages. Only values designed to be public (client id, public key,
   base URL) go in `--dart-define`. If a key must stay secret, it does not belong in the app —
   it needs a backend proxy.
2. **No `!` (bang) on a value that can legitimately be null.** Use pattern matching,
   `switch` expressions, or a sealed state instead. An empty `catch {}` is a bug, not handling.
3. **Widgets render; they never fetch.** No network call, no DB access, no business rule, and
   no side effect inside `build()`. UI → ViewModel/Controller → Repository → Service.
4. **Nothing is disposed automatically.** Every `TextEditingController`, `AnimationController`,
   `FocusNode`, `ScrollController`, `StreamSubscription`, `Timer`, and listener gets a
   `dispose()` / `cancel()` — and `super.dispose()` runs last.
5. **One state-management pattern for the whole app.** Whatever is pinned in §1 is used
   everywhere. Mixing `setState` with a state library in the same screen is a defect.
6. **No hardcoded user-visible strings.** All copy, labels, semantics labels, and error
   messages come from the l10n layer (`lib/l10n/`). No literal Thai/English text in a widget.
7. **No fixed layout sizes.** Use `Expanded`/`Flexible`/`LayoutBuilder`/`MediaQuery.sizeOf`.
   Fixed `width`/`height` only for icons, avatars, dividers, and aspect-ratio boxes.
8. **Types are not validation.** Parse every payload at the boundary (generated models, never
   `as` casts), and validate before use. Runtime data is untrusted until parsed.
9. **A test ships in the same change as the behaviour.** If it cannot be tested, say why
   instead of skipping silently. Test 401/403/empty/error — not only the happy path.
10. **Follow the nearest existing file.** Copying one established pattern beats inventing a new
    one. If no pattern exists, say so and propose one before writing it.

## 3. Where the detailed rules live (read the one that matches the file you touch)

| Touching | Read |
|----------|------|
| `lib/**` layout, layering, routing, DI, offline | `.github/instructions/architecture.instructions.md` |
| any `*.dart` — naming, null safety, dispose, docs, formatting | `.github/instructions/dart.instructions.md` |
| widgets, screens, theming, a11y, responsive | `.github/instructions/flutter-ui.instructions.md` |
| `pubspec.yaml`, adding a package, env/config | `.github/instructions/dependencies.instructions.md` |
| anything security, logging, secrets, permissions, release | `.github/instructions/security.instructions.md` |
| `test/**`, `integration_test/**`, mocks | `.github/instructions/tests.instructions.md` |

## 4. How to work

- **Anything beyond one file: propose a plan first and wait for approval.** No code before the
  plan is accepted. Use the `/plan-first` prompt.
- Adding a feature walks the layers in order: domain entity → data model/service → repository
  → view model → view → wiring → test. Never start from the widget.
- One logical change per commit. Conventional Commits: `feat:`, `fix:`, `refactor:`, `test:`,
  `chore:`.
- `dart analyze` must be clean before you claim you are done; run `dart format .` after edits.
- Finish by stating plainly: what changed, what you verified (the exact command or test that
  proved it), and what you could not verify.

## 5. When unsure

Say so. Do not invent a package name, an API signature, a platform channel, a config key, or a
widget parameter. "I don't know — here is how to check" beats a confident guess. Never claim a
test passed, a build succeeded, or a bug is fixed without running the command that proves it.
