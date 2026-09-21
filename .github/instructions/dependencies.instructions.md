---
applyTo: "pubspec.yaml,pubspec.lock,**/pubspec.yaml,**/pubspec.lock,**/*.json"
---

# Dependencies and configuration (rules 2 + 14)

## Rule 2 — Minimal third-party dependencies

Default answer to "should I add a package?" is **no**. The SD**K first, a package second, your
own implementation last (not first — hand-rolling a bad parser is worse than a maintained
package).

### Decision procedure — answer all five before adding anything

1. **Can the SDK do it?** Check `dart:core`/`dart:async`/`dart:convert`/`dart:io`/`dart:math`
   and the Flutter framework (e.g. `showDatePicker`, `Form`/`TextFormField`, `Image.network`,
   `NavigationBar`, `SelectionArea`, `RestorableProperty`). State what you checked in your reply.
2. **Is it small enough to own?** Under ~50 lines, well-scoped, and stable: write it in
   `core/` and test it. Do not add a package to format a date range.
3. **Is the package healthy?** pub.dev points ≥ 130, published within 12 months, null-safe,
   license compatible (MIT/BSD/Apache-2.0 — flag GPL), ≥ 1 maintainer with a public repo, and a
   real test suite.
4. **What does it drag in?** Run `dart pub deps --style=compact` — count the transitive
   packages and the native/platform channels. A package that pulls 15 more or requires a
   plugin registration per platform needs a strong reason.
5. **What is the exit cost?** If it is abandoned next year, how many files import it? Keep
   third-party types out of `domain/` so it can be swapped behind an interface.

### Standing allowlist — **pinned for this project** (extend deliberately, not per-feature)

เวอร์ชันล่าสุดบน pub.dev ณ 18 ก.ย. 2026 — ใช้ caret range (`^x.y.z`) และอย่าแก้ major เอง
ถ้าจะเปลี่ยนตัวใดตัวหนึ่ง ให้เปลี่ยนที่ `copilot-instructions.md` §1 ด้วย ไม่ใช่แก้ที่ pubspec อย่างเดียว

| Purpose | Pinned | Notes |
|---------|--------|-------|
| State management | **flutter_riverpod `^3.4.3`** | `Notifier`/`AsyncNotifier` + sealed state · **ตัวเดียวทั้งโปรเจกต์** ห้ามผสม provider/bloc/GetX |
| HTTP | **dio `^5.11.1`** | อยู่ใน `core/network/` เท่านั้น ⇒ `domain/` ไม่เคยเห็น dio |
| Model codegen | **freezed `^4.0.2`** + **json_serializable `^6.14.1`** + **build_runner `^2.16.1`** (dev) | generated files ห้ามแก้มือ · ⚠️ freezed 4 ต่างจาก freezed 2 — ห้ามเดา syntax จากความจำ |
| Secure storage | **flutter_secure_storage `^11.2.0`** | tokens/keys เท่านั้น — ห้าม `SharedPreferences` |
| Simple storage | **shared_preferences `^2.5.5`** | ข้อมูลไม่ลับ ไม่วิกฤต |
| Routing | **go_router `^18.0.1`** | typed routes + deep links |
| Localisation | `flutter_localizations` (SDK) + **intl `^0.20.3`** | ต้องมีคู่กับ `gen-l10n` — ทีม Flutter ดูแลเอง |
| Tests | **flutter_test** (SDK) + **mocktail `^1.0.5`** | dev dep · golden ใช้ `matchesGoldenFile` ของ SDK |
| Lints | **flutter_lints `^6.0.0`** | dev dep · ทางเลือกเข้มกว่า: very_good_analysis (ต้องตกลงทั้งทีม) |
| ทางเลือกที่ *ไม่* เลือก | `provider`, `flutter_bloc`, `http`, `equatable`, `hive`, `sqflite`, `auto_route`, `very_good_analysis` | ใช้ได้ แต่ต้องเปลี่ยนที่ constitution ให้ตรงกันทั้งโปรเจกต์ก่อน ไม่ใช่ทยอยเพิ่ม |

Anything outside this table: propose it, with the five answers above, and wait for approval.
Never add a package while implementing a feature "just to make it work".

### Anti-patterns

- A package for one function (`string_validator` to check `isEmpty`).
- Two packages that do the same thing (`http` *and* `dio`; `provider` *and* `riverpod`).
- Copy-pasting an implementation from a package and dropping the dependency — either depend on
  it or cite the license and rewrite it properly.
- A dependency used only by the codegen step in production deps; or a runtime package in
  `dev_dependencies`.
- Editing generated files (`*.g.dart`, `*.freezed.dart`) — fix the source and regenerate.
- `git add pubspec.lock` for applications: an app **must** commit `pubspec.lock` for
  reproducible builds (a package does not).
- Unpinned or `any` constraints. Use caret ranges (e.g. `^5.4.0`) and keep
  `dart pub outdated` clean; never `dependency_overrides` to silence a conflict.

## Rule 14 — Configuration and secrets

- **The app binary is public.** Anyone can unpack it and read every string inside. Therefore:
  only values that are *designed* to be public may live in the app — base URL, client id,
  public analytics key, feature flags.
- **Values that must stay secret never enter the app.** API secrets, signing keys, admin
  tokens, DB credentials, SMS/payment gateways: they belong on a backend that the app calls.
  If a task requires one in the client, stop and say why it cannot be done that way.
- Configuration is injected at build time and read once, through one class:

```dart
// core/config/app_config.dart
/// Build-time configuration. Supplied with
/// `--dart-define-from-file=config/dev.json` (or `--dart-define=KEY=value`).
class AppConfig {
  const AppConfig();

  static const String apiBaseUrl = String.fromEnvironment(
    'API_BASE_URL',
    defaultValue: 'http://localhost:8080',
  );
  static const bool verboseLogs = bool.fromEnvironment('VERBOSE_LOGS');
}
```

- Command shape: `flutter run --dart-define-from-file=config/dev.json`.
  Prefer this over `.env` + `flutter_dotenv` — no extra dependency, and the value is compiled
  rather than shipped as a readable asset.
- The config file per environment (`config/dev.json`, `config/prod.json`) is **git-ignored**;
  commit only `config/example.json` with placeholder values. Never commit a filled config.
- `AppConfig` is the **only** place that reads `String.fromEnvironment`. No scattered
  `String.fromEnvironment` calls, no config passed down through 6 widgets — inject it.
- Never put a secret in: source, an asset, a `const` in Dart, `AndroidManifest.xml`, `Info.plist`,
  a CI log, an error report, an analytics event, or a commit message.
- No default fallback for a *legitimately required* value: if the build is missing
  `API_BASE_URL` in production, fail loudly at startup rather than silently pointing at
  localhost.
- Release builds must not include debug-only endpoints, verbose logging, or a
  `badCertificateCallback => true`. TLS verification is never disabled to make a dev
  environment work — fix the certificate chain instead.
- Secrets in CI come from the platform's secret store (GitHub Actions secrets / env injection),
  never from a committed file. The pre-commit hook and CI secret scan exist to catch the day
  someone forgets.
