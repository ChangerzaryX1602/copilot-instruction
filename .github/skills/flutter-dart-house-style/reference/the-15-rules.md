# The 15 rules — full text with why, examples, and enforcement

Source of truth for the house standard. Each rule: **what** it is, **why** it exists, a
good/bad pair, what enforces it, and how it fails if ignored.
Enforcement labels are honest: ✅ machine-enforced · ~ partial · ✖ convention + reviewer only.

---

## 1. Strict file structure — feature-first, layered inside the feature

**What:** `lib/core/` for cross-cutting code; `lib/features/<feature>/{data,domain,presentation}`.
Dependencies point inward: `presentation → domain ← data`. `domain/` is pure Dart.
**Why:** an AI (and a new teammate) finds the right file by the folder name. Layering makes
testing possible without a device and lets a data source change without touching UI.
**Bad:** `lib/models/user.dart`, `lib/screens/home.dart`, `lib/utils.dart` (a dump folder).
**Good:** `lib/features/call_history/data/repositories/call_history_repository_impl.dart` next to
`domain/repositories/call_history_repository.dart`.
**Enforced by:** ✖ — `architecture.instructions.md` + review. Nothing in the analyzer knows a
layer from a folder.
**Fails as:** 6 months later nobody can change the API client without touching 30 widgets.

## 2. Minimal third-party dependencies — SDK first

**What:** before adding a package: SDK? · under ~50 lines to own? · healthy (≥130 pub points,
published <12 months, license OK)? · transitive cost? · exit cost? Then check the allowlist.
**Why:** every package is code you did not write, a supply-chain path, and a future migration.
On a company app the license question is a legal question.
**Bad:** `flutter_nice_button` for a `TextButton`; `http` *and* `dio` in the same repo.
**Good:** `showDatePicker` from the SDK instead of `date_range_picker_pro`; `String.fromEnvironment`
instead of `flutter_dotenv`.
**Enforced by:** ~ `sort_pub_dependencies`, `depend_on_referenced_packages`, `flutter pub outdated`
in CI — the allowlist itself is a human decision.
**Fails as:** an abandoned package blocks a Flutter upgrade; nobody can remove it because 40 files
import its types.

## 3. Strict null safety and error handling

**What:** no `!` on a value that can legitimately be null (patterns/`switch`/sealed state
instead); typed `on X catch` at the I/O boundary; no empty catch; no catching `Error`; every
`Future` awaited or explicitly handled; no `dynamic`.
**Why:** `!` converts a compile-time obligation into a runtime crash in the user's hand. An empty
catch converts an outage into a silent wrong screen.
**Bad:** `final n = resp.user!.name!; … catch (e) {}`
**Good:** `final name = switch (resp.user) { final User(:final name) => name, null => l10n.unknown };`
and `on TimeoutException catch (e, st) { log.warning(…); return const Result.failure(Failure.timeout()); }`
**Enforced by:** ✅ strict analyzer mode + 15 lints; ✖ for the `!` ban itself (no official lint
exists) — `dart-review` prompt + reviewer.
**Fails as:** a null crash on exactly the response shape nobody tested (empty profile, expired
session).

## 4. `const` and widget extraction

**What:** `const` wherever the analyzer allows; large widgets become separate
`StatelessWidget` classes (not `_buildX()` methods); `build()` short and shallow.
**Why:** a `const` subtree is never rebuilt — free performance. A widget class is isolated,
`const`-able, testable, and shows up in the inspector; a method cannot be any of those.
**Bad:** `Widget _header() => Text(context.l10n.title);` inside a 400-line `build()`.
**Good:** `class _Header extends StatelessWidget { const _Header(); … }`
**Enforced by:** ✅ `prefer_const_constructors*`, `prefer_const_literals_to_create_immutables`,
`avoid_unnecessary_containers`, `sized_box_for_whitespace` · ~ extraction/depth is a review call.
**Fails as:** scroll jank and dropped frames on a mid-range phone; every rebuild touches the whole
tree.

## 5. Separation of concerns — logic out of the UI

**What:** the widget renders state and forwards intent. No network, no DB, no business rule, no
I/O in `build()`. UI → view model → repository → service.
**Why:** it is the only way to test behaviour without a widget, and the only way to reuse it.
**Bad:** `Dio().get('/calls').then((r) => setState(...))` inside `initState`.
**Good:** `switch (viewModel.state) { … }` rendering a sealed state provided by an injected
view model.
**Enforced by:** ~ `no_logic_in_create_state`, `use_build_context_synchronously` · the real check
is review.
**Fails as:** a screen that cannot be tested, breaks offline, and duplicates error handling.

## 6. Dart naming conventions — the standard, not a guess

**What:** files/directories `snake_case`; variables, parameters, functions, fields **and
constants** `lowerCamelCase`; types (`class`/`enum`/`typedef`/`extension`/`mixin`)
`UpperCamelCase`; `_` for private members only.
**Why:** it is the language's convention, it is what every Dart tool and package uses, and the
linter enforces it — deviating guarantees an analyzer warning and a reviewer comment.
**Bad:** `CallHistoryView.dart`, `const DEFAULT_TIMEOUT`, `String UserName`, `IUserRepository`.
**Good:** `call_history_view.dart`, `const defaultTimeout`, `final String userName`,
`abstract class UserRepository`.
**Enforced by:** ✅ `constant_identifier_names`, `camel_case_types`, `non_constant_identifier_names`,
`file_names`, `library_names`, `package_names`, `library_private_types_in_public_api`,
`no_leading_underscores_for_local_identifiers`.
**Fails as:** noise in every code review plus a linter warning people learn to ignore.

## 7. Memory management — dispose everything

**What:** every `TextEditingController`, `AnimationController`, `FocusNode`, `ScrollController`,
`PageController`, `Timer`, `StreamSubscription`, `ValueNotifier`/`ChangeNotifier` is disposed or
cancelled; listeners are removed; `super.dispose()` is last; `mounted` is checked after `await`.
**Why:** leaks are silent until the app is used for 20 minutes, then it is an OOM kill or a
`setState() called after dispose` crash in the store reviews.
**Bad:** `_controller = TextEditingController();` with no `dispose()`, or `super.dispose()` first.
**Good:** the dispose block that cancels the subscription, disposes nodes in reverse creation
order, then calls `super.dispose()`.
**Enforced by:** ✅ `cancel_subscriptions`, `close_sinks`, `avoid_slow_async_io` · ordering and
`mounted` guards are review.
**Fails as:** memory growth per screen visit; a crash when a stream event lands after pop.

## 8. SOLID and DRY — with the rule of three

**What:** one reason to change per class; extend by adding a case or implementation; no subclass
that throws `UnimplementedError`; small interfaces; depend on abstractions. Extract duplication on
the **third** occurrence, or immediately when it is a domain rule.
**Why:** over-abstraction makes code unreadable today; under-abstraction makes a bug live in three
places. The rule of three is the cheap discriminator.
**Bad:** a "generic" `AppCard` with 11 optional parameters used once; a `UserService` that also
does email, caching and analytics.
**Good:** `CallHistoryRepository` (calls only) + `AnalyticsService` injected where needed.
**Enforced by:** ✖ — `dart-review` checklist item 8.
**Fails as:** a fix applied to one of three copies; a "generic" component nobody dares to change.

## 9. Testable code design — constructor injection

**What:** dependencies come in through the constructor; the composition root is `main.dart`;
no hidden singletons or locator lookups inside widgets; inject `Clock`, `Uuid`, `Random`.
Depend on interfaces from `domain/`.
**Why:** you cannot test what the class builds for itself. This rule is what makes the whole test
pyramid possible.
**Bad:** `class CallViewModel { final _repo = CallRepositoryImpl(Dio()); }`
**Good:** `class CallViewModel { CallViewModel({required CallHistoryRepository repository, required Clock clock}) : _repository = repository, _clock = clock; }`
**Enforced by:** ✖ on design · ~ CI enforces the outcome (tests must pass, coverage floor).
**Fails as:** no tests are written because writing them is hard; the first regression ships.

## 10. Concise documentation

**What:** `///` on public API — one-line summary ending with a period, then parameters/returns/
throws where not obvious. Explain **why**, not what. `//` for implementation notes. `TODO(owner)`.
**Why:** the next reader is usually an AI agent with no context; a wrong or missing doc is how it
invents a call that does not exist.
**Bad:** `/// Gets the name` above a getter, or no docs at all on a public repository method.
**Good:** `/// Retries once because the PBX drops the first INVITE after an idle period.`
**Enforced by:** ~ `slash_for_doc_comments`, `comment_references`, `flutter_style_todos`; setting
`public_member_api_docs: true` enforces *presence* (never quality).
**Fails as:** duplicated logic written by the next agent because the intent was undocumented.

## 11. One state-management pattern

**What:** the pattern pinned in `.github/copilot-instructions.md`, used everywhere. `setState`
only for local ephemeral UI state. State exposed to the UI is immutable.
**Why:** two patterns in one screen means two sources of truth and a rebuild bug nobody can
reproduce. It also makes the agent's output unpredictable.
**Bad:** a Bloc feature next to `setState`-managed screens, both fed by the same repository.
**Good:** one view model per screen, sealed state, `setState` reserved for a tab index.
**Enforced by:** ✖ — review, plus the pinned value in layer 2 so the agent cannot pick freely.
**Fails as:** a screen that shows stale data after the other pattern updates first.

## 12. No hardcoded strings

**What:** every user-visible string — labels, buttons, hints, tooltips, snackbars, error messages,
semantics labels, date/number formats — comes from the l10n layer. No concatenated sentences.
**Why:** retrofit localisation is a rewrite; concatenation makes word order untranslatable; and
scattered literals make copy changes a hunt.
**Bad:** `Text('ไม่พบรายการ')`, `Text('สวัสดี ' + name)`, `Text('$count รายการ')`.
**Good:** `Text(context.l10n.noCalls)`, `Text(context.l10n.greeting(name))`, ICU plural in the ARB.
**Enforced by:** ✖ — no standard lint. `dart-review` item 12; genuine enforcement needs a custom
lint pipeline (extra dependency) or a brittle grep gate.
**Fails as:** a second language requested the week before release.

## 13. Responsive and adaptive UI

**What:** no fixed sizes for layout; `Expanded`/`Flexible`/`LayoutBuilder`/`MediaQuery.sizeOf`;
breakpoints in one file; handles 320 dp, tablet, landscape, keyboard open, `textScaler` 2.0, RTL.
**Why:** the phone you test on is never the phone the user has. Text scaling is an accessibility
setting people actually use.
**Bad:** `Container(width: 375, height: 200)`; `MediaQuery.of(context).size.width * 0.8` inline.
**Good:** `LayoutBuilder` + `ConstrainedBox` + `Flexible` around long labels; breakpoints from
`core/theme/breakpoints.dart`.
**Enforced by:** ✖ — adding a widget test at two sizes plus `textScaler` 2.0 is how you turn this
into ~.
**Fails as:** a yellow-and-black overflow stripe on a small phone — the most visible bug there is.

## 14. Security and environment variables

**What:** no credentials in the app. Build-time config through one `AppConfig` reading
`String.fromEnvironment` with `--dart-define-from-file`; the filled config file is git-ignored;
secrets live server-side.
**Why:** the binary is public; a secret in the client is a published secret. `.env` in Flutter
also needs a third-party package, which contradicts rule 2.
**Bad:** `const apiSecret = 'sk_live_…';` or a committed `config/prod.json` with a key.
**Good:** `AppConfig.apiBaseUrl` from `--dart-define`, tokens in `flutter_secure_storage`, secret
scan in CI + pre-commit.
**Enforced by:** ✅ gitleaks in CI + `.githooks/pre-commit` (with a grep fallback so a missing
binary never means "no check").
**Fails as:** a leaked key in the APK, or in git history forever.

## 15. Formatting and trailing commas

**What:** trailing comma on every multi-line argument/parameter list and collection literal;
`dart format` clean; single quotes; ordered imports; no `print` in `lib/`.
**Why:** the trailing comma is the signal `dart format` uses to break a call vertically — it is
what makes a diff one-line-per-argument instead of one 200-character line.
**Bad:** a 260-character `Container(...)` call, or a vertical layout achieved by manual newlines
with no trailing comma.
**Good:** trailing comma after the last argument; let `dart format` place the breaks.
**Enforced by:** ✅ `require_trailing_commas` + CI format check + `formatter.trailing_commas: preserve`.
**Fails as:** unreviewable diffs; formatter churn fighting hand formatting.
