# Tape Mobile

## What this is
.


## Commands
- flutter pub get
- dart run build_runner build --delete-conflicting-outputs
- dart run build_runner watch --delete-conflicting-outputs   (during active work)
- flutter analyze
- flutter test
- flutter run -t lib/main.dart   (flavors not set up yet — see Stack note below)

## Stack
- Riverpod 3, code generation.
- Freezed 3 — abstract required, no exceptions. If this ever looks wrong: guide → Data Layer → Freezed 3 model.
- Dio — custom REST API, not Firestore.
- Flutter flavors — dev / staging / prod, native setup, not flutter_flavorizr. DEFERRED: not set up yet, single lib/main.dart entry point for now. When flavors land: restore the --flavor command above, split into main_dev.dart/main_staging.dart/main_prod.dart, and wire AppUrls.baseUrl's --dart-define per flavor (guide's Network Layer section).

## Architecture — read before writing any feature
@docs/riverpod-state-management-guide.md

Ten-File Formula, every feature, no exceptions:
Model → Repository contract → Mock impl → Provider → Notifier → Widget → Page
→ Router → Real impl → the swap (one line)

Mock before real. Always. Never write a Real*Impl before the mock version
has a working screen in front of it.

## Design source of truth
@docs/design-context.md
@docs/design-plan.md
Per-screen exact specs live in docs/design-specs/<screen>.md — read the one
file for the screen being built. Never skim the whole Figma file at once.

## API contract
@docs/api-contract.md
If a route or field isn't in this file, ask — don't guess the shape.

## Current progress
@docs/build-status.md

## Working rhythm
- One piece at a time. See .agents/skills/next-piece.md — never batch
  multiple screens or steps into a single pass.
- End of session: update docs/build-status.md. See .claude/skills/update-status.md.
  Only check off a step if it's actually working, not just written.

## Conventions
- Every implementing class ends in _impl.dart
- @Riverpod(keepAlive: true) on Dio, repository providers, api providers. Never on notifiers.
- @JsonKey wherever the backend uses snake_case. Skip it for state that never leaves the app.

## Do not
- Edit .g.dart or .freezed.dart files by hand
- Add a package without asking first
- Touch core/ or application/ without flagging it — shared by every feature

# Antigravity Execution Rules
- **Review-Driven Mode:** Use Review-driven mode for all file modifications and commands like `flutter pub add`. Pause for my approval before executing destructive actions.
- **Context Gathering:** Always read the widget tree, running app state, and active routes via the Dart & Flutter MCP server before writing any UI modifications.
- **Figma to Flutter Rules:** Do not invent generic container styling from CSS-like Figma tokens. Extract all SVGs as isolated assets first before building the layout. Follow the `docs/design-specs/<screen>.md` rigidly.
- **Figma extraction scope:** One frame per pass. Never request a whole page or
  multiple screens in a single extraction. If a screen is larger than a card or
  dialog, split it into sections and extract section by section.
- **Auto Layout is a precondition.** Before extracting, confirm the frame and
  every container inside it use Auto Layout. If any child is absolutely
  positioned, stop and flag it — do not translate absolute x/y into
  Stack/Positioned.
- **Tokens must be published.** If the Figma MCP returns raw hex values or pixel
  literals instead of named tokens, the design system is unpublished. Stop and
  flag it — do not invent a palette, and do not carry lunchbari_provider's
  values over without confirmation.
- **Layer names are identifiers.** If layer names are generic (Frame 427,
  Group 12), stop and ask for a rename pass before generating code.
- **Generated code is a draft.** Always follow extraction with a run against the
  Dart & Flutter MCP server: check the widget tree for overflow and check
  runtime errors before marking the step done in build-status.md.