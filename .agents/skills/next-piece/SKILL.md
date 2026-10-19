---
name: next-piece
description: Use when the user says "next" during a build session. Builds one full screen (spec through as much of its Ten-File-Formula checklist as isn't blocked) in a single call, stopping only when that screen is done, blocked, or a real question needs the user's input.
disable-model-invocation: true
---

# Next Piece

0. Progress lives in three files, split by role (2026-08-17):
   `docs/build-status/shared.md` (Foundations, Auth #1-3, Splash), and
   `docs/build-status/provider.md` / `docs/build-status/rider.md` for
   everything else. To find which file a given build-order row belongs in:
   Auth (#1-3) and Splash → `shared.md`. Every other row in
   `docs/design-plan.md`'s build order table is already grouped by role —
   #4-6 are provider work, #7-9 are rider work, then the table's own
   **Phase A — Provider** heading covers #10-29 and **Phase B — Rider**
   covers #30-37. When genuinely unsure, the row's screen name/description
   makes the role obvious. Read/write the ONE file that matches the
   feature currently being built — don't touch the other two.
1. Read the matching build-status file (per step 0) and docs/design-plan.md.
2. If the next feature in the build order has no docs/design-specs/<feature>.md
   yet, capture that ONE screen's spec first — exact layout, colors, states,
   no summarizing. Also add that feature's Ten-File Formula checklist to the
   matching build-status file — ten unchecked boxes: Model, Repository
   contract, Mock impl, Provider, Notifier, Widget, Page, Router, Real impl,
   Swap. Do not stop here — continue straight into step 3 for this same
   feature, within this same call.
3. Find the first unchecked box in that file for the feature currently
   being built. Build that one file, verify it the same way every prior step
   in this codebase has been verified (analyze/lint/test as applicable),
   check it off with the same level of detail prior entries use, and then
   continue straight to the next unchecked box for the SAME feature — still
   within this one call. Do not wait for the user to ask again between
   steps.
3a. If a box is "Real impl" or "Swap", check docs/api-contract.md for that
   feature's route before building it. If the route isn't there yet, skip
   past that box — do not check it off, do not guess at the request or
   response shape — and continue on to the next unchecked box for the same
   feature.
4. Stop and report back when any of these happens:
   - Every box for the current feature is either checked or correctly
     skipped as BLOCKED (Real impl/Swap with no route yet). Report a short
     summary of what got built, then ask whether to continue to the next
     feature — do not start the next feature's spec (step 2) without that
     go-ahead.
   - A genuine open question comes up that only the user can settle — a
     real product ambiguity or a design conflict not already resolved by
     precedent elsewhere in this codebase. Stop immediately and ask, even
     mid-feature, rather than guessing and moving on.
   - Something breaks (a failing analyze/lint/test) that isn't a quick,
     obvious fix.
   One feature fully built (or correctly blocked) per call is the
   checkpoint — not one file per call.