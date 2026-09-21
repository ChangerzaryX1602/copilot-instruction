---
applyTo: "lib/**,test/**,integration_test/**"
---

# Architecture, layering and file structure (rule 1)

**Feature-first, layered inside the feature.** One canonical shape — do not invent a second
one, and do not flatten it "just for this one screen".

```
lib/
├── main.dart                     # runApp() เท่านั้น — ห้ามมี logic
├── app.dart                      # MaterialApp.router, theme, locale, bootstrapping
├── core/                         # ใช้ข้ามทุก feature (ไม่รู้จัก feature ใด ๆ)
│   ├── config/                   # env (String.fromEnvironment), flavors, feature flags
│   ├── error/                    # Failure, Result, exception → failure mapping
│   ├── network/                  # client + interceptors + timeouts
│   ├── router/                   # route table (go_router) + guards
│   ├── theme/                    # design tokens, ThemeData, text styles
│   ├── logging/                  # logger + redaction (ห้าม log PII)
│   └── widgets/                  # shared widgets — StatelessWidget ล้วน
├── features/
│   └── call_history/
│       ├── data/
│       │   ├── models/           # DTO + fromJson/toJson (generated) — เท่านั้นที่แตะ JSON
│       │   ├── datasources/      # remote (API) / local (cache, db)
│       │   └── repositories/     # implement ของ domain interface + cache/retry/mapping
│       ├── domain/
│       │   ├── entities/         # immutable, pure Dart, ไม่รู้จัก JSON และไม่รู้จัก Flutter
│       │   ├── repositories/     # abstract interface (สัญญาที่ data ต้องทำได้)
│       │   └── use_cases/        # เฉพาะ logic ที่ถูกใช้ซ้ำหรือยาวจนรก view model
│       └── presentation/
│           ├── view_models/      # state + intent (ชื่อตาม state mgmt ที่ pin ไว้)
│           ├── views/            # หน้าจอ (route target)
│           └── widgets/          # widget เฉพาะ feature นี้
└── l10n/                         # .arb + ไฟล์ที่ gen-l10n สร้าง (ห้ามแก้มือ)
```

Test mirrors `lib/`: `test/features/call_history/…`, `test/core/…`, plus
`integration_test/` for flows.

## Dependency direction (violating this is a defect, not a preference)

```
presentation ──▶ domain ◀── data          core ──▶ (ไม่มีใคร)
```

- `domain/` is pure Dart: **no `package:flutter`, no JSON, no HTTP, no logging, no DI calls.**
  It may import `dart:core`, `dart:async`, and `meta`.
- `data/` may import `domain/` and `core/`; **never** `presentation/`.
- `presentation/` may import `domain/` and `core/`; it must **not** import `data/` — it talks
  to the abstract repository, not the implementation.
- `core/` never imports a feature. If `core` needs feature knowledge, the abstraction is wrong.
- **No cross-feature imports.** Feature A does not import `features/b/…`. Shared code moves to
  `core/`, or becomes an explicit sub-package of one feature that the other consumes through
  its `domain/` interface.
- Nothing outside `data/models/` parses raw JSON. Nothing outside `data/datasources/` knows an
  endpoint path. Nothing outside `data/repositories/` maps DTO → entity.

## Adding a feature — walk the layers in this order

1. `domain/entities/` — the immutable model the UI actually wants (not the API's shape).
2. `domain/repositories/` — the abstract interface (what the feature needs, nothing more).
3. `data/models/` — DTO + generated `fromJson`/`toJson`.
4. `data/datasources/` — the call that returns DTOs, with timeout and error mapping.
5. `data/repositories/` — implements the interface: cache, retry, DTO → entity mapping.
6. `presentation/view_models/` — state, intents, loading/error/empty handling.
7. `presentation/views/` + `widgets/` — pure rendering of that state.
8. Register in DI. 9. Test. 10. **Then** a route entry if it is a new screen.

Never start from the widget. If asked to "just add a screen", say the layers it needs and do
them in order.

## Dependency injection (supports rule 9)

- Dependencies arrive through the **constructor**. A class that builds its own client,
  repository, or clock is untestable — that is a bug.
- The composition root is `main.dart` / `app.dart` (or one `bootstrap()` function): the only
  place allowed to know concrete implementations.
- No hidden singletons: no `static final instance = …`, no `Get.find()` / locator lookups
  inside widgets or view models. (A DI container that injects into constructors is fine.)
- Inject the non-deterministic things too: `Clock`, `Uuid`, `Random`, `Connectivity` — a test
  must be able to freeze them.

## Routing

- One route table (`core/router/`), typed route definitions, no string-built paths at call
  sites.
- Auth/onboarding gates live in a redirect guard, not scattered `if (loggedIn)` in screens.
- Deep links map to the same destination as the in-app navigation; an unknown/invalid deep
  link goes to a defined fallback, never a crash.
- Pass identifiers (ids), not objects, through routes — so a cold start via deep link works.

## Offline, cache and failure policy

- State it explicitly: is this screen online-only, cache-first, or offline-capable?
- Any cached data has: source, timestamp, TTL, and a visible freshness signal when stale.
- Writes that can fail offline are either blocked with a clear message or queued — never
  silently dropped.
- Retries are bounded, idempotent, and never happen on non-idempotent writes without a key.
- Every remote call has a timeout. A missing timeout is an outage waiting for a slow network.

## Things this file forbids

- A `utils.dart` / `helpers.dart` / `common.dart` dump folder — it is where architecture goes
  to die. Put the code in the layer that owns it, or `core/<topic>/`.
- Business logic in `core/widgets/`.
- A `models/` folder at `lib/` root shared by every feature.
- `part`/`part of` to split a feature across files (use separate libraries and imports).
- God repositories (`AppRepository`) and god view models (`AppViewModel`).
