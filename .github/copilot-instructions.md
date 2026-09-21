# Repository instructions (Flutter / Dart)

> Always-on rules — read on **every** request. Keep this file under one page.
> Anything longer belongs in `.github/instructions/` (path-scoped) or `.github/skills/` (on demand).
>
> **ค่านี้เติมไว้แล้ว** ด้วย stack ที่แนะนำสำหรับโปรเจกต์ที่ยังไม่มีโค้ด (stable ล่าสุด ณ 18 ก.ย. 2026)
> 4 บรรทัดต่อไปนี้คือ "การตัดสินใจของทีม" ไม่ใช่ความจริงสากล: state management · codegen ·
> routing · minimum OS — เปลี่ยนได้ แต่แก้ที่ไฟล์นี้ที่เดียว และห้ามใช้สองแบบผสมกันในโปรเจกต์เดียว
> ยืนยันของจริงบนเครื่องก่อนเริ่มงาน: `flutter --version` แล้วอ่าน `pubspec.yaml`

## 1. Stack — pinned on purpose

Copilot's training data is older than this repo. **Never emit idioms from a different major
version.** If an API was renamed or removed in the pinned version, stop and say so — do not write
the old call, and do not guess the new one from memory.

- Flutter **3.47.5** · Dart **3.13.4** (`environment: sdk: ^3.13.0` in `pubspec.yaml`)
- State management: **flutter_riverpod 3.x** — `Notifier` / `AsyncNotifier` + sealed state.
  **ห้ามผสม**: no `provider`, no `bloc`, no `GetX`, no `setState` for app state.
- HTTP client: **dio 5.x** (interceptors, timeouts, typed errors) — ไม่ใช้ `http` ในโปรเจกต์เดียวกัน
- Models / JSON: **freezed 4.x + json_serializable 6.x + build_runner 2.x** (dev)
  ⚠️ freezed major 4 ใช้ syntax ต่างจาก freezed 2 (`class X with _$X`) — **ห้ามเดา syntax จาก
  ความจำ**: อ่าน major ที่ติดตั้งจริงใน `pubspec.lock` แล้วเปิดเอกสารของ major นั้นก่อนเขียนโมเดล
  หลังแก้โมเดลให้รัน `dart run build_runner build --delete-conflicting-outputs` และ commit ไฟล์
  ที่ gen (`.g.dart`, `.freezed.dart`) เข้า repo
- Routing: **go_router 18.x** — typed routes, deep links, redirect guards
- Storage: **flutter_secure_storage 11.x** (tokens เท่านั้น) · **shared_preferences 2.x** (ข้อมูลไม่ลับ)
- Localisation: `flutter_localizations` (SDK) + **intl 0.20.x** ผ่าน `gen-l10n` (ดู `l10n.yaml`)
- Lints: **flutter_lints 6.x** + `analysis_options.yaml` ของ repo นี้ · Format: `dart format`, `page_width: 100`
- Tests: **flutter_test** + **mocktail 1.x** · golden: ใช้ `matchesGoldenFile` ของ SDK
- Minimum OS: **Android 8.0 (API 26)** · **iOS 14.0** (ถ้าต้องรองรับเครื่องเก่ากว่านี้ ให้แก้ที่
  `android/app/build.gradle.kts` + `ios/Podfile` และเขียนเหตุผลใน PR)

## 2. Non-negotiable rules

1. **No secrets in the app, ever.** A Flutter binary is public — it can be unpacked. No API
   secret, signing key, admin token, or DB credential in source, in a `.env` committed to git,
   in logs, or in commit messages. Only values designed to be public (client id, public key,
   base URL) go in `--dart-define`. If a key must stay secret, it does not belong in the app —
   it needs a backend proxy.
2. **No `!` (bang) on a value that can legitimately be null.** Use pattern matching,
   `switch` expressions, or a sealed state instead. An empty `catch {}` is a bug, not handling.
3. **Widgets render; they never fetch.** No network call, no DB access, no business rule, and
   no side effect inside `build()`. UI → Notifier/ViewModel → Repository → Service.
4. **Nothing is disposed automatically.** Every `TextEditingController`, `AnimationController`,
   `FocusNode`, `ScrollController`, `StreamSubscription`, `Timer`, and listener gets a
   `dispose()` / `cancel()` — and `super.dispose()` runs last.
5. **One state-management pattern for the whole app** — the one pinned in §1 (Riverpod).
   Mixing `setState` with it for app state in the same screen is a defect.
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
  → notifier/view model → view → wiring → test. Never start from the widget.
- One logical change per commit. Conventional Commits: `feat:`, `fix:`, `refactor:`, `test:`,
  `chore:`.
- `dart analyze` must be clean before you claim you are done; run `dart format .` after edits.
- Finish by stating plainly: what changed, what you verified (the exact command or test that
  proved it), and what you could not verify.

## 5. When unsure

Say so. Do not invent a package name, an API signature, a platform channel, a config key, a
widget parameter, or a codegen syntax you are not certain about. "I don't know — here is how to
check" beats a confident guess. Never claim a test passed, a build succeeded, or a bug is fixed
without running the command that proves it.
