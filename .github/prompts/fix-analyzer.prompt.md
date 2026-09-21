---
mode: agent
description: "แก้ error/warning ของ analyzer ให้หมด โดยไม่ปิดกฎหนี"
---

# Fix analyzer findings

Target: `flutter analyze --fatal-infos` and `dart format --set-exit-if-changed` clean.

Rules of engagement:

1. Run the analyzer first and paste the real output; list the distinct findings and count them.
2. **Fix the cause, not the symptom.** Every one of these is forbidden unless you state a
   concrete reason and get approval:
   - `// ignore:` / `// ignore_for_file:` (if truly needed, it must name the *specific* rule
     and carry a same-line explanation — a bare ignore is a finding)
   - adding `dynamic`, `late`, or `!` to silence a null-safety error
   - widening a type (`Object?`, `Map<String, dynamic>`) to dodge `strict-casts`
   - deleting or commenting out the failing code
   - editing generated files (`*.g.dart`, `*.freezed.dart`)
3. Classify each finding: real bug · wrong design · style · obsolete rule. Say which, then fix.
4. When a `prefer_const_*` fires, only add `const` if the value is genuinely compile-time
   constant — never "fix" it by making the value dynamic-friendly.
5. When `use_build_context_synchronously` fires, add the `mounted` / `context.mounted` guard;
   never move the `await` to dodge the lint.
6. When `unawaited_futures` / `discarded_futures` fires, decide: await it, return it, or
   explicitly `unawaited(...)` with a comment on why ignoring the result is safe.
7. Prefer `dart fix --apply` for mechanical issues — then review the diff file by file and
   explain anything non-obvious it changed. Never run it blind and commit.
8. Re-run the analyzer after each batch and report the before/after counts.
9. If a lint rule is genuinely wrong for this repo, do not delete it silently: propose the
   change to `analysis_options.yaml` with the reason, and wait for approval.

Finish with: the commands run, the real before/after finding counts, files changed, and any
rule you believe deserves to change in `analysis_options.yaml`.
