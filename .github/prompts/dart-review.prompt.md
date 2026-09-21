---
mode: agent
description: "รีวิวโค้ด Dart/Flutter ที่เพิ่งเขียน ตาม 15 กฎ + checklist จริง"
---

# Review this change against the house rules

Review the change below. Be adversarial and specific — your job is to find what is wrong, not to
approve. **Do not edit code in this turn**; produce findings only.

Change to review: `<paste diff / list files / "the working tree changes">`

## Output format

For every finding, in severity order:

```
[BLOCKER|MAJOR|MINOR] file.dart:line — rule N (name)
what is wrong (one sentence)
why it matters (the failure this causes, not a style preference)
the fix, as a concrete code suggestion
```

Then a final table: `rule → count of findings`, and an explicit list of rules you checked and
found **clean**. If you found nothing, say what you inspected and how you know — "looks good" is
not a review.

## Checklist (walk all of it, in this order)

1. **Structure** — file in the right layer? dependency direction respected (domain pure, no
   `data/` import from presentation, no cross-feature import)? `utils.dart` dump created?
2. **Dependencies** — new package? was the SDK checked first, is it on the allowlist, what does
   it pull in? `pubspec.lock` committed?
3. **Null safety & errors** — any `!` that is not locally provable? any empty `catch`, any catch
   without a type, any swallowed error, any `throw` of a non-`Exception`, any unawaited future,
   any `catch (Error)`?
4. **`const` & extraction** — missing `const` the analyzer did not catch? `_buildX()` methods or
   widget-returning functions that should be classes? nesting deeper than 4? a dynamic list
   built with `Column(children: …map())`?
5. **Logic vs UI** — I/O or business rules inside `build()`, a widget, or a `State`? navigation
   decided by a domain rule in the view? formatting rules inline?
6. **Naming** — `snake_case` file, `lowerCamelCase` identifiers and constants (not
   `SCREAMING_CAPS`), `UpperCamelCase` types, no type/`I`-prefix names, booleans read as
   assertions?
7. **Dispose** — every controller, focus node, subscription, timer, listener created → disposed
   or cancelled, `super.dispose()` last, `mounted` guards after `await`?
8. **SOLID & DRY** — a class doing two jobs? duplication appearing for the third time? an
   abstraction invented for a single use? a subclass that throws `UnimplementedError` (LSP)?
9. **Testability** — dependencies injected through the constructor, or constructed internally?
   `DateTime.now()` / `Random()` called directly? can this be tested without a network?
10. **Docs** — `///` on public API, summary ending with a period, parameters/returns where not
    obvious, `//` used for implementation notes only, TODOs with an owner, docs that match the
    code?
11. **State management** — one pattern only, no `setState` for app state, immutable state
    exposed, view model disposed?
12. **Strings** — any user-visible literal, any concatenated sentence, any plural/date built by
    hand, any un-localised error/semantics label?
13. **Responsive** — fixed sizes for layout, hardcoded breakpoints, overflow risk at 320 dp /
    landscape / `textScaler` 2.0, missing `SafeArea`, missing empty/loading/error state,
    accessibility (labels, 48 dp, colour-only signals)?
14. **Secrets** — any credential, token, key, or real hostname in code/config; a filled config
    file about to be committed; tokens not in secure storage; PII in logs, crash reports, or
    analytics?
15. **Formatting** — trailing commas present, `dart format` clean, no hand-wrapping, single
    quotes, import order, no `print`.

Finish with the three things you would fix first if only three could be fixed, and one thing
you are unsure about (say so rather than guessing).
