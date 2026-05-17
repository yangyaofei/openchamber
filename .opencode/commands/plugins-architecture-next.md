---
description: Implement the next OpenChamber plugin architecture task
agent: build
---

You are working in the OpenChamber repository.

Goal: continue implementing the plugin architecture host platform itself.

Execution mindset:

- This command is for implementation, not planning or commentary.
- Do not shrink the task because it looks large. The selected task is the scope.
- Do not stop after analysis, discovery, or a proposed plan. Carry the selected task through code changes, docs/status updates, and validation.
- Do not defer obvious substeps to a future agent when they are part of the selected task's checklist or acceptance criteria.
- Resolve blockers yourself when they are within the selected task scope. Only mark blocked when continuing would require a missing prerequisite, a product decision, or unrelated architectural work outside the selected task.
- Implement the whole selected task to its acceptance criteria. Keep changes correct and behavior-preserving, but do not use minimalism as an excuse to leave required checklist items undone.
- If the task has multiple checklist items, work through all of them unless one is genuinely blocked; do not cherry-pick the easiest subset.
- Keep going until the task is either completed and validated, or explicitly blocked with a concrete reason in the task notes and final response. If blocked, also propose the concrete fix path and offer to implement it next; if the fix is within the selected task scope, implement it instead of stopping.

Read first:

1. docs/plugins-architecture.md
2. docs/plugins-implementation.md

Do not use arguments. Determine the next task yourself.

Task selection:

- Find the first task in @docs/plugins-implementation.md with `Status: [ ]` whose dependencies are completed or explicitly marked as already available.
- Prefer strict task order unless a task is blocked.
- If a task is blocked, update its status to `[!]`, add a concise blocker note, and select the next unblocked task only if doing so is safe.
- Do not select tasks from @docs/plugins-integration.md. This command is only for implementing the architecture host platform.

Execution rules:

- Change the selected task status to `[~]` before coding.
- Implement only the selected task.
- Do not migrate internal features unless the selected architecture task explicitly includes a vertical slice.
- Follow @docs/plugins-architecture.md as the source of truth.
- Keep changes minimal and preserve existing behavior.
- Enforce policy in host/server logic, not only UI.
- Preserve web, Electron, and VS Code behavior, or document intentional differences in the task notes.
- Update the selected task checklist, acceptance criteria, and notes in @docs/plugins-implementation.md.
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
- State the next recommended task ID.
