---
name: update-status
description: Use when the user says "update status" or at the end of a session. Marks completed steps in the right docs/build-status/*.md file based on what was actually built.
disable-model-invocation: true
---

# Update Status

1. Look at what was actually created or edited this session.
2. Match each file against its feature's Ten-File Formula checklist, in the
   build-status file that feature belongs to: `docs/build-status/shared.md`
   (Auth #1-3, Splash), `docs/build-status/provider.md`, or
   `docs/build-status/rider.md` — see `.claude/skills/next-piece`'s step 0
   for how to tell which.
3. Check off exactly the steps with a real, working file behind them —
   not steps that were only discussed.
4. Do not check off "Real impl" or "Swap" unless the API call was
   actually tested against a real response, not just written.