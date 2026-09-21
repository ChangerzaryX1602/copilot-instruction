---
applyTo: "**"
---

# Security, privacy and logging (rule 14, applied to every file)

Applies to all files, in every layer. On the client, "secure" means: **we assume the binary and
the network are hostile, and we never keep what we do not need.**

## 1. Nothing secret ships in the app

- The app binary is public. `strings app.apk` is a 5-second exercise; obfuscation raises effort,
  not secrecy.
- Allowed in the app: base URL, client id, public keys, feature flags.
- Never in the app: API secrets, DB credentials, signing keys, admin/privileged tokens,
  third-party server-side keys (payment, SMS, maps with billing). Those live on a backend the
  app calls. If a task seems to require one, say so and propose the backend-proxy design instead
  of hardcoding it.
- Never in source, assets, `AndroidManifest.xml`, `Info.plist`, `const` literals, tests,
  fixtures, commit messages, logs, or crash reports.

## 2. Tokens and local storage

- Access/refresh tokens: `flutter_secure_storage` only (Keychain / Android Keystore). Never
  `SharedPreferences`, never a plain file, never a `sqflite` column, never a log line.
- Refresh tokens rotate; a transient failure must not wipe the session. A hard 401 wipes it and
  routes to login — in one place, not per screen.
- Anything cached that is personal (profile, call history, messages) is cleared on logout, and
  is not written to a world-readable location (`getTemporaryDirectory`, not external storage).
- Screenshots/recents preview: mask sensitive screens
  (`FLAG_SECURE` on Android, snapshot-protection on iOS) for screens showing personal data.
- Never log or display the full token, even truncated — an id prefix is enough for support.

## 3. Logging: redact before it leaves the device

- Use the app's logging layer (`core/logging/`), never `print` (`avoid_print`).
- Verbose/debug logs are compiled out or gated by `AppConfig.verboseLogs` — false in release.
- **Never log**: tokens, `Authorization` headers, request/response bodies that can contain
  credentials, phone numbers, MSISDN, SIP URIs, email/address, precise location, device
  identifiers, or free-text user input.
- Log identifiers that exist *for* correlation instead: an opaque request/call/session id.
- Strip these from HTTP interceptors **and** crash reporters — the reporter is an outbound
  network call, so it is subject to the same rules as the API.
- An error message shown to a user never contains a stack trace, SQL, an internal hostname, or
  a status code alone ("เกิดข้อผิดพลาด" + a reference id beats "DioException 500").

## 4. Network

- HTTPS only, including dev (a self-signed cert is a config problem to fix, never
  `badCertificateCallback => true`).
- Certificate/public-key pinning for a high-value API, with a documented rotation plan; a pin
  without a rotation plan is an outage with extra steps.
- Every request has a timeout; every retry is bounded and idempotent (no blind retry of a POST
  that creates something without an idempotency key).
- Validate and bound anything from the network before using it in a file path, a URL, a
  `MethodChannel` call, a `WebView`, or a platform intent — client-side trust of server data is
  still a bug.
- WebView: no `JavaScriptMode.unrestricted` on remote content without a strict allowlist; never
  expose a JS bridge to untrusted pages.
- Deep links are untrusted input: validate scheme/host/path/params before acting, and never let
  a deep link trigger a privileged action without an auth check.

## 5. Platform permissions

- Ask **when used**, in context, with a rationale the user can read — never on app start, never
  all at once.
- Handle all three outcomes: granted, denied, permanently denied (open settings), plus
  restricted-by-policy.
- The app must stay usable when permission is refused: degrade the feature, do not dead-end.
- Declare only the permissions actually used in `AndroidManifest.xml` / `Info.plist`; every
  extra one is a review flag and a store-rejection risk.

## 6. Personal data

- Collect the minimum fields needed for the feature; do not log or persist "just in case".
- Any personal data stored or transmitted has a stated retention and a delete path (logout,
  account deletion, or a documented TTL) — for a company app, follow the company's PDPA policy
  and name the data owner in the code comment or PR description.
- Analytics events carry no personal data: no raw phone numbers, no email, no free text, no
  user content. Hashed, non-reversible identifiers at most.
- A screenshot, log dump, or debug bundle attached to a bug report is a data export — strip it
  first.

## 7. Release and build hardening

- Separate flavours/config for dev, staging, prod. A release build never points at a dev host.
- Release: obfuscate + split debug symbols (`--obfuscate --split-debug-info`), keep the symbol
  map in a private store for crash de-obfuscation.
- No debug tooling in release: no `--inspect`, no HTTP inspector, no hidden debug menu, no
  `debugShowCheckedModeBanner`, no test backdoor endpoint.
- `flutter pub outdated` + `dart pub audit`-equivalent scan in CI; a known-vulnerable dependency
  is a blocking finding.
- Signing keys and store credentials are never in the repo, and never on a machine without
  full-disk encryption.

## 8. Do not fake security work

- Do not claim a screen is "secure" because it is hidden, obfuscated, or requires a client-side
  check. Authorization is decided by the server.
- Do not silently disable a security control (TLS verification, hostname checks, R8 rules,
  permission checks) to unblock a build. Say what is blocking, and propose the real fix.
- If asked to implement something in this file's territory and the safe way is more work, say
  so plainly and do the safe version.
