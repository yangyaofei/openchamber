---
description: Migrate the next internal feature to the plugin architecture
agent: build
---

You are working in the OpenChamber repository.

Goal: continue migrating existing internal OpenChamber features onto the plugin architecture.

Execution mindset:

- This command is for implementation, not planning or commentary.
- Do not shrink the task because it looks large. The selected task is the scope.
- Do not stop after analysis, discovery, or a proposed plan. Carry the selected task through code changes, docs/status updates, and validation.
- Do not defer obvious substeps to a future agent when they are part of the selected task's checklist or acceptance criteria.
- Resolve blockers yourself when they are within the selected task scope. Only mark blocked when a required plugin host primitive is missing, a product decision is required, or continuing would require unrelated architecture work.
- Implement the whole selected task to its acceptance criteria. Prefer registering existing code as built-in plugin contributions before moving files, but do not use minimalism as an excuse to leave required checklist items undone.
- If the task has multiple checklist items, work through all of them unless one is genuinely blocked; do not cherry-pick the easiest subset.
- Keep going until the task is either completed and validated, or explicitly blocked with a concrete reason in the task notes and final response. If blocked, also propose the concrete fix path and offer to implement it next; if the fix is within the selected task scope, implement it instead of stopping.

Read first:

1. docs/plugins-architecture.md
2. docs/plugins-integration.md

Do not use arguments. Determine the next task yourself.

Task selection:

- Find the first task in @docs/plugins-integration.md with `Status: [ ]` whose dependencies are completed or already available in the codebase.
- This command assumes the plugin host primitives already exist. Do not implement missing architecture primitives here.
- If a required primitive is missing, mark the selected task `[!]`, add a concise blocker note naming the missing primitive, and stop.
- Do not select tasks from @docs/plugins-implementation.md. This command is only for migrating internal features and product surfaces.

Execution rules:

- Change the selected task status to `[~]` before coding.
- Implement only the selected integration task.
- Prefer registering existing code as built-in plugin contributions before physically moving files.
- Preserve current behavior when the feature is enabled.
- Implement disabled-feature behavior if the selected task requires it.
- Do not expose raw Zustand stores, raw `RuntimeAPIs`, Electron IPC, VS Code APIs, or mutable `opencodeClient` to plugin code.
- Preserve server-side validation and safety gates.
- Preserve lazy-loading boundaries for heavy UI surfaces.
- Preserve web, Electron, and VS Code parity, or document intentional differences in the task notes.
- Update the selected task checklist, acceptance criteria, and notes in @docs/plugins-integration.md.
- Mark the task `[x]` only after implementation and validation are complete.

Validation:

- Run `bun run type-check`.
- Run `bun run lint`.
- If validation fails, fix failures that are within the selected task scope.
- If validation cannot be fixed within scope, leave the task `[!]` with a blocker note and report the failure.

Final response:

- State the selected task ID and title.
- Summarize changed files.
- Report validation results.
- State the next recommended integration task ID.
