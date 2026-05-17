# OpenChamber Plugin Architecture Implementation Backlog

Status: full architecture implementation backlog draft

References:

- Architecture source of truth: `docs/plugins-architecture.md`
- Internal feature migration backlog: `docs/plugins-integration.md`

## Purpose

This document is the task backlog for implementing the plugin architecture itself. It is not the backlog for migrating every internal feature. Internal feature migration lives in `docs/plugins-integration.md` and must only start after the required architecture tasks here are complete.

Agents should use this file as the executable checklist for building the host platform: plugin types, registries, feature gates, UI contribution runtime, server contribution runtime, built-in loader, diagnostics, external bundled plugin support, auth provider API, and runtime server plugin support.

## How To Use This Document

- Read `docs/plugins-architecture.md` before starting any task.
- Pick exactly one task whose dependencies are complete.
- Change that task status to `[~]` while working.
- Implement only that task.
- Update the task checklist, acceptance criteria, and notes before finishing.
- Change the task status to `[x]` only after implementation and validation are complete.
- Run `bun run type-check` and `bun run lint` before marking implementation tasks complete.
- Do not start internal feature migrations from `docs/plugins-integration.md` until this file says the required foundation tasks are complete.

## Status Legend

- `[ ]` Not started
- `[~]` In progress
- `[x]` Completed
- `[!]` Blocked
- `[-]` Cancelled or superseded

## Agent Prompt Template

```md
You are working on the OpenChamber plugin architecture.

Read first:
1. docs/plugins-architecture.md
2. docs/plugins-implementation.md
3. docs/plugins-integration.md

Implement only this architecture task:

<TASK_ID>

Rules:
- Verify this task's dependencies are completed in docs/plugins-implementation.md.
- Do not migrate internal features unless this task explicitly says so.
- Follow docs/plugins-architecture.md as the source of truth.
- Keep changes minimal and preserve current behavior.
- Enforce policy in host/server logic, not only UI.
- Preserve web/Electron/VS Code parity or document intentional differences.
- Update this task's status/checklist/notes in docs/plugins-implementation.md.
- If the task unblocks integration tasks, mention them in the notes.
- Run bun run type-check and bun run lint before final response.
```

## Milestone Overview

1. `PLUG-IMPL-001` to `PLUG-IMPL-005`: foundational host architecture.
2. `PLUG-IMPL-006` to `PLUG-IMPL-010`: initial vertical slices and UI registry opening.
3. `PLUG-IMPL-011` to `PLUG-IMPL-015`: settings, commands, tool renderers, storage, external bundled plugins.
4. `PLUG-IMPL-016` to `PLUG-IMPL-018`: auth providers, runtime server plugins, hardening/docs.

Recommended strict order for the foundation:

1. `PLUG-IMPL-001`
2. `PLUG-IMPL-002`
3. `PLUG-IMPL-003`
4. `PLUG-IMPL-004`
5. `PLUG-IMPL-005`

After `PLUG-IMPL-005`, agents may start the first integration tasks in `docs/plugins-integration.md`, beginning with terminal/files/git, but only if their listed dependencies are complete.

## Architecture Implementation Tasks

### PLUG-IMPL-001: Core Plugin Types And Registry

Status: [ ]

Depends on:

- None

Unblocks:

- `PLUG-IMPL-002`
- `PLUG-IMPL-003`
- `PLUG-IMPL-004`
- `docs/plugins-integration.md` `PLUG-FOUNDATION-001`

Current files:

- New plugin registry/type files to be created.

Scope:

- Introduce plugin definitions and a central registry without changing UI/server behavior yet.

Implementation checklist:

- [ ] Add shared plugin type definitions.
- [ ] Add `definePlugin(def)` helper.
- [ ] Define `PluginDefinition`, `PluginManifest`, `RuntimePluginEntry`, `PluginSource`, `PluginTarget`, and `PluginCapability`.
- [ ] Define contribution record types for UI, server, commands, settings, tools, models, storage, and lifecycle.
- [ ] Add a plugin registry module that can register plugins and capture setup errors.
- [ ] Add plugin source ordering: `builtin`, `bundled`, `user`.
- [ ] Add deterministic ordering by `priority`, `pluginId`, and `contributionId`.
- [ ] Add duplicate plugin ID detection.
- [ ] Add disposable registration handling.
- [ ] Add diagnostics representation for plugin load state, granted capabilities, denied capabilities, setup errors, and contribution counts.
- [ ] Add tests for registry ordering, duplicate plugin ID rejection, setup error capture, and disposal.

Acceptance criteria:

- [ ] A dummy built-in plugin can register without affecting app behavior.
- [ ] Duplicate IDs fail deterministically.
- [ ] Setup errors are recorded and do not crash the whole app unless the plugin is marked required.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- Keep this task free of feature migrations.
- Prefer a small stable core over a broad first implementation.

### PLUG-IMPL-002: Feature And Capability Registry

Status: [ ]

Depends on:

- `PLUG-IMPL-001`

Unblocks:

- `PLUG-IMPL-003`
- `PLUG-IMPL-004`
- Feature-gated integration tasks in `docs/plugins-integration.md`

Current files:

- `packages/ui/src/stores/useFeatureFlagsStore.ts`
- `packages/ui/src/App.tsx`
- `packages/web/server/index.js`
- New feature registry files to be created.

Scope:

- Make feature enablement host-owned and visible to UI and server.

Implementation checklist:

- [ ] Replace one-off feature state with namespaced feature IDs.
- [ ] Preserve existing plan mode behavior while moving to feature registry semantics.
- [ ] Define feature IDs for first internal migrations:
  - `openchamber.feature.terminal`
  - `openchamber.feature.files`
  - `openchamber.feature.git`
  - `openchamber.feature.plan-mode`
- [ ] Add server-side feature snapshot source.
- [ ] Add `GET /api/features` or include feature snapshot in plugin diagnostics.
- [ ] Hydrate UI feature state during startup.
- [ ] Provide leaf selectors such as `isFeatureEnabled(featureId)`.
- [ ] Add server-side helper for gating feature-owned routes.
- [ ] Add unknown feature fallback behavior.
- [ ] Add tests for feature normalization, unknown feature fallback, and disabled feature route behavior.

Acceptance criteria:

- [ ] UI can query feature enablement by namespaced ID.
- [ ] Server can gate routes by feature ID.
- [ ] Plan mode still works.
- [ ] Unknown feature IDs do not crash the app.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- Feature state should not become a broad hot Zustand store.
- This task should not migrate terminal/files/git yet.

### PLUG-IMPL-003: UI Contribution Runtime

Status: [ ]

Depends on:

- `PLUG-IMPL-001`
- `PLUG-IMPL-002`

Unblocks:

- `PLUG-IMPL-005`
- `PLUG-IMPL-006`
- `PLUG-IMPL-007`
- `PLUG-IMPL-008`
- UI feature migrations in `docs/plugins-integration.md`

Current files:

- New UI plugin runtime files to be created.
- Later consumers include `MainLayout.tsx`, `RightSidebarTabs.tsx`, `SettingsView.tsx`, and `CommandPalette.tsx`.

Scope:

- Add generic UI contribution primitives and host components.

Implementation checklist:

- [ ] Implement `ctx.ui.fill(slotId, component, options)`.
- [ ] Implement `ctx.ui.surface(surfaceId, config)`.
- [ ] Implement `ctx.ui.replace(targetId, component, options)`.
- [ ] Implement `ctx.ui.wrap(targetId, wrapper, options)`.
- [ ] Add `Slot` component for rendering fills.
- [ ] Add `SurfaceOutlet` component for rendering registered surfaces by placement or active surface ID.
- [ ] Add `ReplaceableSurface` component for replacement targets with fallback content.
- [ ] Add wrapper composition helper for `wrap` targets.
- [ ] Add plugin render error boundaries.
- [ ] Add runtime, feature, and capability filters to contribution resolution.
- [ ] Preserve support for lazy components in surface configs.
- [ ] Add tests for fill ordering, replacement conflict detection, wrappers, disabled feature filtering, missing component fallback, and diagnostics.

Acceptance criteria:

- [ ] A dummy built-in UI plugin can add visible content to a test slot.
- [ ] A dummy built-in UI plugin can register a surface.
- [ ] Replacement conflicts are deterministic and diagnosable.
- [ ] Plugin render errors do not crash the whole app.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- Use stable product surface IDs from `docs/plugins-architecture.md`, not component names.
- Do not migrate major views in this task except for tiny test/demo contributions if needed.

### PLUG-IMPL-004: Server Contribution Runtime

Status: [ ]

Depends on:

- `PLUG-IMPL-001`
- `PLUG-IMPL-002`

Unblocks:

- `PLUG-IMPL-005`
- Server-side feature migrations in `docs/plugins-integration.md`

Current files:

- `packages/web/server/index.js`
- `packages/web/server/lib/opencode/bootstrap-runtime.js`
- `packages/web/server/lib/opencode/feature-routes-runtime.js`
- New server plugin runtime files to be created.

Scope:

- Add server plugin route/middleware/lifecycle registration while preserving current ordering.

Implementation checklist:

- [ ] Implement server plugin registry integration.
- [ ] Implement `ctx.server.routes(id, register, options)`.
- [ ] Implement middleware phase registration.
- [ ] Implement lifecycle hooks: `beforeRoutes`, `afterRoutes`, `beforeListen`, `afterListen`, `beforeShutdown`.
- [ ] Define explicit route phases based on `docs/plugins-architecture.md`.
- [ ] Add feature and capability gates for server contributions.
- [ ] Add plugin diagnostics endpoint or extend diagnostics payload.
- [ ] Add tests for route ordering, disabled route behavior, setup error handling, lifecycle disposal, and auth/proxy/static fallback ordering.

Acceptance criteria:

- [ ] Existing routes behave the same before any built-in route migration.
- [ ] A dummy protected route can be registered by a plugin.
- [ ] A disabled plugin does not register its routes.
- [ ] Type-check passes where applicable.
- [ ] Lint passes.

Notes:

- Preserve these ordering rules: auth routes before `/api` gate, feature routes before OpenCode proxy, proxy before static fallback, static SPA fallback last.
- WebSocket handlers must remain path-filtered and auth-checked.

### PLUG-IMPL-005: Built-In Plugin Loader

Status: [ ]

Depends on:

- `PLUG-IMPL-003`
- `PLUG-IMPL-004`

Unblocks:

- `PLUG-IMPL-006`
- `PLUG-IMPL-007`
- `PLUG-IMPL-008`
- First feature migrations in `docs/plugins-integration.md`

Current files:

- New built-in plugin entrypoints to be created.
- Existing app/server startup files to wire loader.

Scope:

- Register first-party built-in plugins through a central built-in plugin list.

Implementation checklist:

- [ ] Add UI built-in plugin list.
- [ ] Add server built-in plugin list.
- [ ] Add startup setup flow that creates plugin contexts and registers contributions.
- [ ] Add runtime target filtering.
- [ ] Add required and optional capability grants.
- [ ] Add plugin enablement config hooks.
- [ ] Add diagnostics for built-ins.
- [ ] Add tests that built-ins load in deterministic order.
- [ ] Add at least one no-op built-in plugin to prove the path.

Acceptance criteria:

- [ ] Built-in plugins register through the same API shape intended for bundled plugins.
- [ ] Disabled built-ins contribute nothing.
- [ ] Diagnostics show enabled/disabled/error state.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- This is the final foundation task before vertical slices.

### PLUG-IMPL-006: Terminal Vertical Slice

Status: [ ]

Depends on:

- `PLUG-IMPL-005`
- `docs/plugins-integration.md` terminal dependencies satisfied or mirrored here

Unblocks:

- Broader workbench surface migration
- `docs/plugins-integration.md` `PLUG-FEATURE-TERMINAL-001`

Current files:

- `packages/ui/src/components/views/TerminalView.tsx`
- `packages/ui/src/components/layout/BottomTerminalDock.tsx`
- `packages/web/server/lib/terminal/runtime.js`

Scope:

- Prove the architecture with a feature that has UI surfaces and server runtime ownership.

Implementation checklist:

- [ ] Create `openchamber.plugin.terminal` built-in plugin definition.
- [ ] Register `TerminalView` as a UI surface with placements `workbench.main` and `workbench.bottom-dock`.
- [ ] Make the bottom dock render the terminal surface through the plugin registry.
- [ ] Register terminal server routes/runtime through server plugin contribution or adapter.
- [ ] Gate terminal UI and routes behind `openchamber.feature.terminal`.
- [ ] Keep VS Code terminal unavailable/stub behavior explicit.
- [ ] Add disabled UI and disabled route tests.

Acceptance criteria:

- [ ] Terminal appears in the same places when enabled.
- [ ] Terminal disappears and routes are unavailable when disabled.
- [ ] Existing terminal behavior remains intact when enabled.
- [ ] Diagnostics show terminal contributions.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- This task may update `docs/plugins-integration.md` task `PLUG-FEATURE-TERMINAL-001` if it fully satisfies that migration.

### PLUG-IMPL-007: Files Vertical Slice

Status: [ ]

Depends on:

- `PLUG-IMPL-005`
- `PLUG-IMPL-006` recommended

Unblocks:

- File/context renderer migration
- `docs/plugins-integration.md` `PLUG-FEATURE-FILES-001`

Current files:

- `packages/ui/src/components/views/FilesView.tsx`
- `packages/ui/src/components/layout/SidebarFilesTree.tsx`
- `packages/web/server/lib/fs/routes.js`
- `packages/web/server/lib/fs/search.js`

Scope:

- Prove a file/editor surface and filesystem server routes can be plugin-owned.

Implementation checklist:

- [ ] Create `openchamber.plugin.files` built-in plugin definition.
- [ ] Register `FilesView` as a `workbench.main` surface.
- [ ] Register sidebar files tree as a `workbench.right-panel` surface or slot contribution.
- [ ] Register filesystem server routes through server plugin contribution.
- [ ] Gate files UI and routes behind `openchamber.feature.files`.
- [ ] Preserve VS Code filesystem bridge behavior.
- [ ] Add disabled-feature tests and persistence fallback tests.

Acceptance criteria:

- [ ] Files view and right-panel files tree behave as before when enabled.
- [ ] Disabled files feature removes UI navigation and gates filesystem routes according to policy.
- [ ] Persisted active tab/context references to files fall back safely.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- Files writes/deletes/renames are high-risk operations and must remain host/server validated.

### PLUG-IMPL-008: Git Vertical Slice

Status: [ ]

Depends on:

- `PLUG-IMPL-005`
- `PLUG-IMPL-006` recommended
- `PLUG-IMPL-007` recommended

Unblocks:

- GitHub migration
- Git settings migration
- `docs/plugins-integration.md` `PLUG-FEATURE-GIT-001`

Current files:

- `packages/ui/src/components/views/GitView.tsx`
- `packages/ui/src/components/views/git/*`
- `packages/ui/src/components/layout/RightSidebarTabs.tsx`
- `packages/web/server/lib/git/routes.js`
- `packages/web/server/lib/git/service.js`

Scope:

- Prove a complex feature with UI surfaces, server routes, commands/settings hooks, and side-effect behavior.

Implementation checklist:

- [ ] Create `openchamber.plugin.git` built-in plugin definition.
- [ ] Register `GitView` as a `workbench.main` surface and `workbench.right-panel` surface.
- [ ] Move right sidebar Git tab registration to plugin surface/tab registry.
- [ ] Register Git server routes through server plugin contribution.
- [ ] Register Git-related command palette actions if command registry exists.
- [ ] Register Git settings page/sections if settings registry exists.
- [ ] Gate Git UI/routes behind `openchamber.feature.git`.
- [ ] Preserve right-sidebar Git polling or move it into plugin activation lifecycle.
- [ ] Add disabled-feature tests and Git route ordering tests.

Acceptance criteria:

- [ ] Git view behavior is unchanged when enabled.
- [ ] Disabled Git removes UI surfaces and gates Git routes.
- [ ] No hardcoded `git` right sidebar tab is required in the migrated path.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- Git writes and identity handling are high-risk.

### PLUG-IMPL-009: Open Core UI Registries

Status: [ ]

Depends on:

- `PLUG-IMPL-006`
- `PLUG-IMPL-007`
- `PLUG-IMPL-008`

Unblocks:

- Most UI migrations in `docs/plugins-integration.md`

Current files:

- `packages/ui/src/stores/useUIStore.ts`
- `packages/ui/src/components/layout/MainLayout.tsx`
- `packages/ui/src/components/layout/RightSidebarTabs.tsx`
- `packages/ui/src/components/layout/ContextPanel.tsx`
- `packages/ui/src/components/views/SettingsView.tsx`
- `packages/ui/src/components/ui/CommandPalette.tsx`

Scope:

- Reduce major hardcoded registration hotspots after the first vertical slices.

Implementation checklist:

- [ ] Replace `MainTab` closed union with registered surface IDs plus core constants.
- [ ] Add persisted active surface migration and fallback.
- [ ] Replace `RightSidebarTab` closed union with registered right-panel surface IDs.
- [ ] Add persisted right-panel tab migration and fallback.
- [ ] Replace or bridge `ContextPanelMode` closed union with renderer registry.
- [ ] Start moving settings pages to settings page registry if ready.
- [ ] Start moving command palette actions to command registry if ready.
- [ ] Add tests for unknown/missing persisted IDs.

Acceptance criteria:

- [ ] Unknown/missing plugin surface IDs do not break persisted UI state.
- [ ] Core navigation can render registered surfaces without hardcoded switches for migrated features.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- This task is broad. If it becomes too large, split it into main tabs, right panel, context panel, settings, and command palette subtasks.

### PLUG-IMPL-010: Plugin Diagnostics Surfaces

Status: [ ]

Depends on:

- `PLUG-IMPL-005`

Unblocks:

- External bundled plugin support
- Runtime server plugins

Current files:

- Diagnostics endpoint/UI files to be determined.

Scope:

- Make plugin state inspectable for agents and maintainers.

Implementation checklist:

- [ ] Add `GET /api/plugins`.
- [ ] Add `GET /api/plugins/:pluginId` if useful.
- [ ] Include plugin ID, source, targets, enabled state, capabilities, contributions, setup errors, replacement conflicts, and storage usage if available.
- [ ] Add UI diagnostics surface in settings or a dev panel.
- [ ] Add tests for diagnostics output.

Acceptance criteria:

- [ ] Agents can inspect loaded plugins without reading runtime internals.
- [ ] Disabled/error plugins are visible.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- Diagnostics are critical once multiple agents work independently.

### PLUG-IMPL-011: Settings And Command Registries

Status: [ ]

Depends on:

- `PLUG-IMPL-003`
- `PLUG-IMPL-004` for server settings schemas

Unblocks:

- Settings-heavy feature migrations
- Command palette/native menu/slash command migrations

Current files:

- `packages/ui/src/components/views/SettingsView.tsx`
- `packages/ui/src/lib/settings/metadata.ts`
- `packages/ui/src/components/ui/CommandPalette.tsx`
- `packages/ui/src/hooks/useMenuActions.ts`
- `packages/ui/src/components/chat/CommandAutocomplete.tsx`

Scope:

- Add host-level registries for settings pages/sections and commands/actions.

Implementation checklist:

- [ ] Add settings page registration API.
- [ ] Add settings section registration API.
- [ ] Add server settings schema registration API.
- [ ] Add command registration API.
- [ ] Add command palette group/search provider API.
- [ ] Add shortcut/native menu adapter path.
- [ ] Migrate at least one low-risk settings page or section.
- [ ] Migrate at least one command palette action.
- [ ] Add persisted settings page fallback tests.

Acceptance criteria:

- [ ] A plugin can register a settings page/section.
- [ ] A plugin can register a command palette action.
- [ ] Existing settings and command behavior remains intact.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- Settings validation must remain server-enforced.

### PLUG-IMPL-012: Tool Renderer Registry

Status: [ ]

Depends on:

- `PLUG-IMPL-003`

Unblocks:

- Tool renderer migrations
- Chat/tool plugin integrations

Current files:

- `packages/ui/src/components/chat/message/parts/ToolPart.tsx`
- `packages/ui/src/components/chat/message/toolRenderers.tsx`
- `packages/ui/src/components/chat/message/parts/toolPresentation.tsx`
- `packages/ui/src/components/chat/message/parts/toolRenderUtils.ts`
- `packages/ui/src/lib/toolHelpers.ts`

Scope:

- Introduce stable tool rendering APIs while protecting chat hot paths.

Implementation checklist:

- [ ] Add tool metadata registry.
- [ ] Add exact renderer registration.
- [ ] Add wildcard/prefix renderer registration.
- [ ] Add icon registry.
- [ ] Add classifier registry for expandable/static/standalone presentation.
- [ ] Add output language detector registry.
- [ ] Add side-effect hint registry.
- [ ] Migrate built-in OpenCode tool renderers to built-in tool plugin.
- [ ] Preserve existing render output and expansion behavior.
- [ ] Add memoization boundaries and tests for renderer lookup.
- [ ] Add performance safeguards for streaming updates.

Acceptance criteria:

- [ ] Existing tool rendering is unchanged.
- [ ] A test plugin can register a renderer for a custom tool name.
- [ ] Renderer lookup is deterministic and cheap.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- This is a high hot-path risk task.

### PLUG-IMPL-013: Plugin Storage And Settings Schema

Status: [ ]

Depends on:

- `PLUG-IMPL-001`
- `PLUG-IMPL-004`
- `PLUG-IMPL-011` recommended

Unblocks:

- External bundled plugins
- User/runtime server plugins

Current files:

- Zustand persistence utilities.
- Server settings runtime files.
- New storage facade files to be created.

Scope:

- Provide safe durable plugin settings and state.

Implementation checklist:

- [ ] Implement plugin-scoped global storage.
- [ ] Implement plugin-scoped workspace storage.
- [ ] Add schema version and migration callbacks.
- [ ] Add byte/count quota limits.
- [ ] Add uninstall cleanup API.
- [ ] Add sensitive data policy and redaction rules.
- [ ] Add server-side settings schema contribution model.
- [ ] Add tests for migration, quota, missing plugin, and cleanup.

Acceptance criteria:

- [ ] Plugins can persist namespaced state safely.
- [ ] Unknown plugin settings do not corrupt core settings.
- [ ] Settings validation remains server-enforced.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- Do not let plugins write arbitrary localStorage keys.

### PLUG-IMPL-014: Build-Time Bundled External Plugins

Status: [ ]

Depends on:

- `PLUG-IMPL-005`
- `PLUG-IMPL-010`
- `PLUG-IMPL-013` recommended

Unblocks:

- External plugin authoring

Current files:

- Build scripts/config to be determined.

Scope:

- Prove external plugin packaging without runtime UI code loading.

Implementation checklist:

- [ ] Define plugin config shape for bundled plugin packages.
- [ ] Add generation script for `generated-plugins.ts` or equivalent.
- [ ] Add Vite/web build import integration.
- [ ] Add server import integration for bundled server plugins.
- [ ] Ensure React is shared/peer, not duplicated.
- [ ] Add example demo plugin that fills a UI slot and registers a command.
- [ ] Add example demo server plugin that registers a protected route.
- [ ] Add diagnostics for bundled plugins.
- [ ] Document external bundled plugin authoring constraints.
- [ ] Add build/type-check validation.

Acceptance criteria:

- [ ] A configured bundled plugin can be imported at build time.
- [ ] The plugin can add UI contribution and server route.
- [ ] The plugin is visible in diagnostics.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- Runtime remote UI JavaScript remains out of scope.

### PLUG-IMPL-015: Runtime Capability Facade

Status: [ ]

Depends on:

- `PLUG-IMPL-001`
- `PLUG-IMPL-002`

Unblocks:

- External plugin API hardening
- Runtime parity work

Current files:

- `packages/ui/src/lib/api/types.ts`
- `packages/ui/src/hooks/useRuntimeAPIs.ts`
- `packages/web/src/api/*`
- `packages/vscode/webview/api/*`
- `packages/electron/preload.mjs`

Scope:

- Add narrow plugin host facade above `RuntimeAPIs`.

Implementation checklist:

- [ ] Define runtime capability descriptors.
- [ ] Wrap file APIs by capability.
- [ ] Wrap Git APIs by capability.
- [ ] Wrap terminal APIs by capability.
- [ ] Wrap notifications/editor/VS Code/desktop APIs by capability where applicable.
- [ ] Add runtime target checks.
- [ ] Prevent raw bridge exposure to plugin contexts.
- [ ] Add denied capability tests.

Acceptance criteria:

- [ ] Plugin receives only granted API slices.
- [ ] VS Code/Electron/web differences are explicit.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- This is required before exposing APIs to untrusted or external plugins.

### PLUG-IMPL-016: Auth Provider API

Status: [ ]

Depends on:

- `PLUG-IMPL-004`
- `PLUG-IMPL-010`
- `PLUG-IMPL-013` recommended

Unblocks:

- SSO plugin work

Current files:

- `packages/web/server/lib/ui-auth/ui-auth.js`
- `packages/web/server/lib/opencode/core-routes.js`
- `packages/web/server/lib/opencode/tunnel-auth.js`
- `packages/ui/src/components/auth/SessionAuthGate.tsx`

Scope:

- Prepare for SSO without hardcoding one provider.

Implementation checklist:

- [ ] Define `auth.provider` contribution contract.
- [ ] Refactor existing UI password/passkey/tunnel auth into provider-like internal adapters where feasible.
- [ ] Add login provider discovery endpoint.
- [ ] Update `SessionAuthGate` to render provider-driven login choices.
- [ ] Add callback/status/logout flow contract.
- [ ] Ensure host owns session storage and request verification.
- [ ] Add fake/dev auth provider for tests.
- [ ] Add security tests for auth bypass, callback errors, and disabled provider behavior.

Acceptance criteria:

- [ ] Existing password/passkey behavior still works.
- [ ] A fake auth provider can add a login flow through the provider API.
- [ ] Auth enforcement remains server-side.
- [ ] Type-check passes where applicable.
- [ ] Lint passes.

Notes:

- This is the architectural path for SSO.

### PLUG-IMPL-017: Runtime Server Plugins

Status: [ ]

Depends on:

- `PLUG-IMPL-004`
- `PLUG-IMPL-010`
- `PLUG-IMPL-013`
- `PLUG-IMPL-014` recommended

Unblocks:

- Allowlisted server-only user plugins

Current files:

- Server plugin loader to be created.

Scope:

- Support allowlisted server-only runtime plugins.

Implementation checklist:

- [ ] Define allowlist config for local ESM server plugins.
- [ ] Add loader with path validation.
- [ ] Add capability validation.
- [ ] Add setup error isolation.
- [ ] Add shutdown disposal.
- [ ] Add diagnostics.
- [ ] Add example local server plugin.
- [ ] Add tests for denied path, denied capability, setup failure, route registration, and shutdown cleanup.

Acceptance criteria:

- [ ] Server-only plugin can be loaded from an allowlisted local path.
- [ ] Invalid plugins fail safely.
- [ ] No public routes are exposed accidentally.
- [ ] Type-check passes where applicable.
- [ ] Lint passes.

Notes:

- Runtime UI plugin loading remains out of scope.

### PLUG-IMPL-018: Hardening, Documentation, And CI Invariants

Status: [ ]

Depends on:

- `PLUG-IMPL-010`
- `PLUG-IMPL-014` recommended
- `PLUG-IMPL-017` recommended

Unblocks:

- Stable external plugin authoring

Current files:

- Docs, tests, CI scripts as needed.

Scope:

- Make the platform maintainable and safe for future contributors and agents.

Implementation checklist:

- [ ] Write public plugin authoring guide for build-time bundled plugins.
- [ ] Document supported surface IDs and capability IDs.
- [ ] Add plugin diagnostics UI in settings or dev panel if not already done.
- [ ] Add architectural tests for route order and registry ordering.
- [ ] Add performance checklist for hot-path plugin surfaces.
- [ ] Add security review checklist for new capabilities.
- [ ] Update module docs if plugin architecture becomes mandatory for new features.
- [ ] Update release/build docs for bundled plugin generation.
- [ ] Add CI/test coverage for core plugin invariants where feasible.

Acceptance criteria:

- [ ] New feature work has a documented path to register as a plugin/contribution.
- [ ] Developers and agents can inspect plugins and diagnose contribution conflicts.
- [ ] CI validation covers core plugin invariants.
- [ ] Type-check passes.
- [ ] Lint passes.

Notes:

- This task should be revisited after each major plugin architecture phase.

## Relationship To Internal Integration Backlog

The architecture tasks in this file build the host platform. The tasks in `docs/plugins-integration.md` migrate concrete internal features onto that platform.

Use this rough mapping:

| Architecture task | Integration tasks unblocked |
|---|---|
| `PLUG-IMPL-001` | `PLUG-FOUNDATION-001` |
| `PLUG-IMPL-002` | `PLUG-FOUNDATION-002`, feature gates |
| `PLUG-IMPL-003` | UI surface/slot/wrap/replace migrations |
| `PLUG-IMPL-004` | Server route/runtime migrations |
| `PLUG-IMPL-005` | All built-in plugin migrations |
| `PLUG-IMPL-006` | `PLUG-FEATURE-TERMINAL-001` |
| `PLUG-IMPL-007` | `PLUG-FEATURE-FILES-001` |
| `PLUG-IMPL-008` | `PLUG-FEATURE-GIT-001` |
| `PLUG-IMPL-011` | Settings, commands, slash command tasks |
| `PLUG-IMPL-012` | Tool renderer tasks |
| `PLUG-IMPL-016` | Auth/SSO tasks |

When an architecture task fully completes the matching integration task, update both documents. If it only creates the platform primitive, leave the integration task open.
