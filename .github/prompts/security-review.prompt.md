---
mode: agent
description: "ตรวจ security/privacy ของโค้ดที่แตะ network, storage, log, permission"
---

# Security review of this change

Scope: `<files or diff>` — review against `.github/instructions/security.instructions.md`.
Report findings only; do not edit code in this turn.

## Findings format

```
[CRITICAL|HIGH|MEDIUM|LOW] file.dart:line — category
what is exposed or trustable-by-mistake
the realistic attack or failure (concrete: who does what, and what they get)
the fix
```

Label anything you cannot verify as **UNVERIFIED** rather than assuming it is safe. If a control
lives server-side and you cannot see it, say "not verifiable from the client" instead of
"secure".

## Check each of these

1. **Secrets** — any API secret, token, signing key, password, or privileged credential in
   source, assets, manifest/plist, tests, fixtures, CI config, or commit messages? Anything that
   must be secret but is being shipped in the client (it needs a backend proxy)?
2. **Storage** — tokens in `flutter_secure_storage` (not `SharedPreferences`/plain file/sqflite)?
   personal data cleared on logout? nothing sensitive in a world-readable directory?
3. **Logging & crash reporting** — tokens, auth headers, bodies, phone numbers, SIP URIs, email,
   location, device ids, or free-text input reaching a log, a crash reporter, or analytics? debug
   logging compiled out of release?
4. **Network** — HTTPS everywhere, TLS verification intact (no `badCertificateCallback => true`),
   timeouts present, retries bounded and idempotent, pinning where warranted (with a rotation
   plan)?
5. **Input trust** — server/deep-link/intent data validated and bounded before being used in a
   file path, URL, `WebView`, `MethodChannel` call, or SQL query? JS bridges not exposed to
   untrusted pages?
6. **Permissions** — requested in context with a rationale, all denial outcomes handled, app
   usable when refused, only used permissions declared?
7. **Personal data** — minimum necessary collected, retention and delete path stated, analytics
   free of personal data, no raw user content in events?
8. **Release config** — flavours separated, no dev host or debug tooling in release, obfuscation
   + symbol map stored privately, dependency vulnerability scan clean?
9. **Faked security** — client-side-only authorization, a "hidden" screen treated as protected,
   a security control disabled to unblock a build, an obfuscation claim used as a security
   claim?

## Close with

- The **top 3** to fix before merge, and whether each is a blocker.
- What this review did **not** cover (server-side auth, backend validation, infrastructure,
  third-party SDK behaviour) — a client review is a first pass, not an audit.
- A one-line honest confidence statement.
