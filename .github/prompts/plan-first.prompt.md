---
mode: agent
description: "ห้ามเขียนโค้ดก่อน — ให้วางแผนและรออนุมัติก่อนทุกครั้ง"
---

# Plan first, then code

You are working in a Flutter/Dart repo that follows the house rules in
`.github/copilot-instructions.md`. **Do not write or edit code in this turn.**

## Step 1 — Understand before proposing

1. Restate the task in one sentence, and list what is explicitly **out of scope**.
2. Search the repo for the nearest existing example of the same thing (same layer, same
   pattern) and name the file. If none exists, say so — that changes the plan.
3. State the assumptions you are making about the stack: pinned Flutter/Dart version, state
   management, HTTP client, routing, storage — as read from `pubspec.yaml` / the pinned
   instructions. If a fact is missing, ask for it instead of guessing.

## Step 2 — Plan

Produce a plan with these sections, in this order:

1. **Files** — exact paths, each marked `new` / `edit` / `generated` + one line of purpose.
2. **Order of work** — the layer walk (domain → data → repository → view model → view →
   wiring → test). Nothing starts from the widget.
3. **Interfaces** — the public signatures you intend to add or change (classes, methods,
   parameters, return types). Show real Dart signatures, not prose.
4. **Data flow** — where the data comes from, how it is parsed, where it is cached, what
   happens on failure, and what the four UI states are.
5. **Dependencies** — packages added? If yes: which, which chart in the allowlist, what was
   checked in the SDK first, and what the exit cost is. If a new package is not on the allowlist,
   ask before assuming.
6. **Tests** — which unit/widget tests will be added or changed, and which error path each one
   covers.
7. **Risks and unknowns** — what could break, what you cannot verify from here, and the
   smallest change that would de-risk it.
8. **Commands to verify** — the exact commands you will run when the code exists.

## Step 3 — Stop and wait

End your response with:

> **Plan ready — approve, adjust, or reject? I will not write code until you answer.**

If the user approves, implement in the stated order, one logical change per commit, and run the
verification commands before claiming anything works. If a step of the plan turns out to be
wrong mid-implementation, stop and report the deviation rather than improvising silently.
