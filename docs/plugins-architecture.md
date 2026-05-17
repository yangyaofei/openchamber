# OpenChamber Plugin Architecture

Status: draft architecture inventory

Related documents:

- `docs/plugins-implementation.md`: full phased implementation plan for introducing the plugin platform.
- `docs/plugins-integration.md`: task backlog for migrating internal OpenChamber features onto the plugin architecture.

## Purpose

OpenChamber should become a host platform where product capabilities are registered through a stable plugin/contribution API. The goal is not to add isolated escape hatches for individual UI components. The goal is to define durable extension surfaces, capability gates, lifecycle hooks, and runtime boundaries that allow both built-in features and future external plugins to integrate through the same model.

The primary architectural principle is:

> Layout owns regions. Plugins own capabilities. Contributions bind capabilities into regions.

For example, Git is not a right-sidebar plugin. Git is a capability that may register server routes, commands, settings, tool side-effect hints, and UI surfaces that can be mounted in the main workbench, the right panel, or another compatible region.

## Goals

- Make existing product features register as built-in plugins/contributions.
- Support programmatic enable/disable of internal capabilities such as Git, terminal, files, plan mode, GitHub, skills, MCP, voice, notifications, and scheduled tasks.
- Provide generic UI extension primitives instead of component-specific APIs.
- Provide generic server extension primitives with explicit route/middleware/lifecycle ordering.
- Preserve cross-runtime behavior across web, Electron desktop, and VS Code.
- Keep safety and correctness enforcement in host/server logic, not only in UI plugins.
- Avoid direct plugin access to broad internal Zustand stores, raw Electron IPC, raw VS Code APIs, or mutable OpenCode clients.
- Support build-time bundled external plugins first, then server runtime plugins, and only later consider runtime-loaded UI plugins.

## Non-Goals For The First Iterations

- Arbitrary remote UI JavaScript loading.
- Marketplace installation and updates.
- Full plugin sandboxing.
- Third-party Electron main-process plugins.
- Third-party VS Code extension-host plugins.
- Making every React component a plugin.
- Letting plugins monkey-patch internal stores or React component files.

## Terminology

| Term | Meaning |
|---|---|
| Host | OpenChamber core runtime that owns bootstrap, plugin loading, security policy, registries, shared layout, runtime bridges, and diagnostics. |
| Plugin | A capability package with a manifest and `setup(ctx)` function. Can be built-in, bundled, or user-provided in later phases. |
| Built-in plugin | Internal OpenChamber feature registered through the plugin API but shipped as part of the app. |
| Contribution | A declaration registered by a plugin: UI slot fill, surface, wrapper, route, command, renderer, setting page, etc. |
| Surface | A product-level view/panel/page that can be mounted in compatible regions. |
| Slot | A stable insertion point owned by the host or a surface. Multiple plugins can fill a slot. |
| Replace target | A stable host region whose entire content can be replaced by one selected contribution. |
| Wrapper target | A stable host region that can be wrapped by one or more plugin wrappers. |
| Capability | Permission-like declaration describing which host APIs a plugin requires or may optionally use. |
| Target | Runtime environment where a plugin entrypoint can execute: UI, server, VS Code extension host, Electron main. |

## Current Architecture Observations

### Server

The server already has several good plugin-platform preconditions:

- `packages/web/server/index.js` is the main composition root.
- `createBootstrapRuntime(...)` wires base routes and auth/access routes.
- `createFeatureRoutesRuntime(...)` in `packages/web/server/lib/opencode/feature-routes-runtime.js` already centralizes many feature route registrations.
- Feature modules already exist under `packages/web/server/lib/*` for Git, GitHub, filesystem, terminal, notifications, TTS, quota, skills, scheduled tasks, preview, tunnels, and OpenCode integration.
- Electron desktop runs the web server in-process, so server plugins should naturally apply to desktop.

The main server problem is that feature ownership is still implicit and statically wired. Route ordering, auth gates, OpenCode proxy ordering, SSE compression exclusions, and WebSocket upgrade handlers are sensitive and must become explicit plugin phases.

### UI

The UI has large product boundaries but static registration:

- `MainLayout.tsx` hardcodes main tabs and lazily imports `PlanView`, `GitView`, `DiffView`, `TerminalView`, `FilesView`, and settings windows.
- `RightSidebarTabs.tsx` hardcodes `git`, `files`, and `context` tabs.
- `useUIStore.ts` has closed unions for `MainTab`, `RightSidebarTab`, and `ContextPanelMode`.
- `SettingsView.tsx` and `lib/settings/metadata.ts` are close to registry-style behavior but still rely on static page IDs and hardcoded rendering paths.
- `CommandPalette.tsx` contains hardcoded command/action grouping and search behavior.
- Chat message/tool rendering is centralized but hardcoded in `ToolPart.tsx`, `toolPresentation.tsx`, `toolRenderUtils.ts`, and related files.

The main UI problem is that product surfaces are implemented as direct imports and switches instead of stable extension surfaces.

### Runtime Boundaries

The shared UI receives `RuntimeAPIs` through `window.__OPENCHAMBER_RUNTIME_APIS__`, `RuntimeAPIProvider`, and registry helpers.

Current runtime split:

- Web uses HTTP/WebSocket/SSE routes from `packages/web/src/api/*`.
- Electron loads the web UI from the in-process server and exposes native integrations through preload shims and IPC gates.
- VS Code uses a webview bridge and extension-host proxy. It overrides `fetch` for `/api/*`, implements local bridge operations, and has stricter CSP/asset constraints.

The plugin API must sit above `RuntimeAPIs`. Plugins should not receive raw runtime APIs wholesale. They should receive a capability-scoped host facade.

## Plugin Shape

The plugin definition should use a single setup entrypoint.

```ts
export interface PluginDefinition {
  id: string;
  name: string;
  version: string;
  description?: string;
  source?: PluginSource;
  targets?: PluginTarget[];
  capabilities?: PluginCapability[];
  optionalCapabilities?: PluginCapability[];
  setup(ctx: PluginContext): void | Promise<void>;
}

export type PluginSource = 'builtin' | 'bundled' | 'user';

export type PluginTarget = 'ui' | 'server' | 'vscode-extension-host' | 'electron-main';

export function definePlugin(def: PluginDefinition): DefinedPlugin {
  const setup = (ctx: PluginContext) => def.setup(ctx);
  return Object.assign(setup, { __definition: def });
}
```

The setup entrypoint should register contributions through host APIs. The host should record all registered contributions for diagnostics.

```ts
export default definePlugin({
  id: 'openchamber.git.core',
  name: 'Git',
  version: OPENCHAMBER_VERSION,
  source: 'builtin',
  targets: ['ui', 'server'],
  capabilities: ['ui.surface', 'ui.command', 'server.route', 'git.read', 'git.write'],
  setup(ctx) {
    ctx.ui.surface('git.status', {
      title: 'Git',
      placements: ['workbench.main', 'workbench.right-panel'],
      render: GitView,
    });

    ctx.commands.register('git.commit', { title: 'Commit', run: commitAction });
    ctx.server?.routes('openchamber.git.core.routes', registerGitRoutes);
  },
});
```

## Plugin Context

The plugin context should be stable, capability-filtered, and versioned.

```ts
export interface PluginContext {
  React: typeof React;
  manifest: RuntimePluginEntry;
  capabilities: GrantedCapabilities;
  app: PluginAppAPI;
  hooks: PluginHooksAPI;
  ui: PluginUIAPI;
  commands: PluginCommandsAPI;
  settings: PluginSettingsAPI;
  storage: PluginStorageAPI;
  tools: PluginToolsAPI;
  models: PluginModelsAPI;
  runtime: PluginRuntimeAPI;
  server?: PluginServerAPI;
}
```

Important rules:

- UI plugins should use `ctx.React` rather than importing React directly when loaded externally.
- Built-in plugins may still share direct imports during early migration, but the public ABI should not depend on duplicate React.
- Plugins should not receive raw Zustand stores.
- Plugins should not receive raw `opencodeClient`.
- Plugins should not receive raw Electron or VS Code bridges.
- Host APIs should be disposable where possible.

## Generic UI Primitives

The UI system should expose generic primitives. These are the core of the architecture.

### `fill`

Adds content to a host-owned slot. Multiple plugins may fill the same slot.

```ts
ctx.ui.fill('toolbar.actions', ToolbarBadge, { priority: 20 });
```

Rules:

- Slots are stable product IDs, not React component names.
- Fill ordering is deterministic by source and priority.
- Fill components are mounted under host providers and error boundaries.
- Hot-path slots must isolate plugin render cost.

### `surface`

Registers a view/panel/page that can be mounted in compatible regions.

```ts
ctx.ui.surface('terminal.shell', {
  title: 'Terminal',
  placements: ['workbench.main', 'workbench.bottom-dock'],
  render: TerminalView,
});
```

Rules:

- A surface is not tied to one concrete layout region.
- The host decides which placements are supported in each runtime.
- Surfaces should preserve lazy-loading boundaries.
- Disabled plugins remove their surfaces from navigation and persisted state must fall back safely.

### `replace`

Replaces ownership of an entire host region.

```ts
ctx.ui.replace('workbench.right-panel', CustomRightPanel);
```

Rules:

- Replacement is exclusive by default.
- If multiple plugins register a replacement for the same target, config must explicitly select the active replacement.
- Replacement is for ownership transfer, not small decorations.
- Host shell responsibilities such as resize handles, drag regions, safe areas, and auth gates should remain host-owned unless the target explicitly includes shell replacement.

### `wrap`

Wraps a host surface while preserving the original content.

```ts
ctx.ui.wrap('workbench.left-sidebar', (props, Original) => {
  return <PluginFrame><Original {...props} /></PluginFrame>;
});
```

Rules:

- Wrappers are powerful and should require explicit capability grants.
- Wrapper order must be deterministic.
- Wrappers must not assume DOM structure inside `Original`.

### `renderer`

Registers custom renderers for tools, message parts, files, diffs, settings items, or domain entities.

```ts
ctx.tools.registerRenderer('bash', BashToolRenderer);
ctx.tools.registerRenderer('name_*', PrefixToolRenderer);
```

Rules:

- Renderer lookup must be deterministic.
- Hot-path renderers must be memoized and isolated.
- Renderer registration should be static during plugin setup, not mutated during streaming.

## Capability Model

Capabilities must be explicit and should gate host APIs.

Suggested capability groups:

| Capability | Meaning |
|---|---|
| `ui.fill` | Add content to slots. |
| `ui.surface` | Register views/panels/pages. |
| `ui.replace` | Replace host-owned regions. |
| `ui.wrap` | Wrap host-owned regions. |
| `ui.renderer` | Register renderers. |
| `ui.command` | Register commands/actions. |
| `settings.page` | Register settings pages/sections. |
| `settings.schema` | Register durable settings schema. |
| `storage.global` | Use plugin-scoped global storage. |
| `storage.workspace` | Use plugin-scoped workspace storage. |
| `fs.read` | Read files through host facade. |
| `fs.write` | Write files through host facade. |
| `fs.exec` | Execute filesystem-related commands. High risk. |
| `git.read` | Read Git state. |
| `git.write` | Mutate Git state. |
| `terminal` | Create/use terminal sessions. High risk. |
| `notifications` | Show notifications or contribute notification behavior. |
| `auth.provider` | Register authentication provider. High risk. |
| `server.route` | Register HTTP routes. |
| `server.middleware` | Register middleware. High risk. |
| `server.lifecycle` | Register startup/shutdown hooks. |
| `server.event` | Observe or emit host events. |
| `model.policy` | Filter/rank/decorate models. |
| `desktop.window` | Desktop window integration. Electron-specific. |
| `vscode.command` | Execute VS Code commands. VS Code-specific and high risk. |

Capabilities should be evaluated against:

- Plugin source: built-in, bundled, user.
- Runtime target: web, desktop, VS Code.
- User/workspace configuration.
- Workspace trust or equivalent policy, especially in VS Code.
- Feature enablement.

## Source And Ordering

Plugin source order should be deterministic:

1. `builtin`
2. `bundled`
3. `user`

Within a source, plugins should order by explicit `priority`, then by `id` as a stable tie-breaker.

Contribution ordering should also be deterministic:

```ts
type ContributionOrder = {
  source: 'builtin' | 'bundled' | 'user';
  priority: number;
  pluginId: string;
  contributionId: string;
};
```

For exclusive replacements, priority is not enough. The active replacement should be selected by config:

```json
{
  "plugins": {
    "ui": {
      "replacements": {
        "workbench.right-panel": "company.custom-workbench"
      }
    }
  }
}
```

## UI Extension Surface Inventory

The following stable IDs should describe product surfaces, not implementation files.

| Stable ID | Current implementation | Contribution types | Built-in owner candidates | Notes |
|---|---|---|---|---|
| `app.root` | `App.tsx`, `VSCodeApp.tsx`, mini-chat roots | wrap | `core.workbench` | Runtime-specific roots must remain explicit. |
| `app.providers` | `main.tsx`, `App.tsx` providers | fill, wrap | `core.workbench` | Provider ordering is fragile. |
| `app.overlays` | dialogs, toasts, update overlay | fill | notifications, updates | Mounted near root. |
| `workbench.layout.root` | `MainLayout.tsx` | wrap, replace | `core.workbench` | Desktop/mobile differ significantly. |
| `workbench.main` | `MainLayout.tsx` main area | surface, replace | chat, git, files, terminal, diff, plan | Replaces hardcoded `activeMainTab` switch. |
| `workbench.left-sidebar` | `Sidebar`, `SessionSidebar` | fill, replace, wrap | sessions, projects | Entire current sidebar can become `core.sessions-sidebar`. |
| `workbench.right-panel` | `RightSidebar`, `RightSidebarTabs` | fill, surface, replace, wrap | git, files, project notes | Mobile currently renders Git directly and must be unified. |
| `workbench.bottom-dock` | `BottomTerminalDock` | surface, replace | terminal | Should become generic bottom dock. |
| `workbench.context-panel` | `ContextPanel` | surface, renderer, replace | files, diff, plan, chat, browser | Closed `ContextPanelMode` must open up. |
| `workbench.header` | `Header.tsx` | fill, wrap | workbench, quota, mcp, github | Very large component with many controls. |
| `header.actions` | `Header.tsx` buttons/actions | fill, command | workbench, updates, services | Must preserve desktop drag-region safety. |
| `toolbar.actions` | Header and chat/workbench toolbars | fill | plugin-specific | General slot for badges/buttons. |
| `sessions.sidebar.header` | `SidebarHeader.tsx` | fill, command | sessions, projects, multirun | Current hardcoded project/session actions. |
| `sessions.sidebar.project-list` | `SidebarProjectsList.tsx` | renderer, fill | projects | Hot-ish tree/list. |
| `sessions.sidebar.activity-sections` | `SidebarActivitySections.tsx` | fill, renderer | sessions | Streaming/session activity sensitive. |
| `sessions.row.actions` | `SessionNodeItem.tsx` | fill, command | sessions, folders, pinning | Must preserve row memoization. |
| `sessions.row.badges` | `SessionNodeItem.tsx` | fill | sessions, git, scheduled tasks | Hot path. |
| `sessions.row.context-menu` | row/context menu code | command | sessions | Needs command registry. |
| `sessions.sidebar.footer` | `SidebarFooter.tsx` | fill, command | settings, help, updates | Current footer is hardcoded. |
| `chat.surface` | `ChatView`, `ChatContainer` | wrap, replace | chat | Embedded/mini-chat variants need constraints. |
| `chat.viewport.before` | chat viewport | fill | plugin-specific | Hot path warning. |
| `chat.viewport.after` | chat viewport | fill | plugin-specific | Hot path warning. |
| `chat.input` | `ChatInput.tsx` | fill, wrap, replace | chat-input | Large mixed component. |
| `chat.input.toolbar.left` | input toolbar | fill, command | voice, attachments, models | Mobile differs. |
| `chat.input.toolbar.right` | input toolbar | fill, command | submit, queue, voice | Mobile differs. |
| `chat.input.autocomplete-providers` | file/agent/skill/command autocomplete | renderer, command | files, agents, skills, commands | Needs ordered provider registry. |
| `chat.message` | `ChatMessage.tsx` | wrap, renderer | chat | Hot path and custom comparator. |
| `chat.message.header` | `MessageHeader.tsx` | fill | chat, models | Sticky/mobile variants. |
| `chat.message.actions` | message actions | fill, command | chat | Hot path. |
| `chat.message.body` | `MessageBody.tsx` | renderer, wrap | chat | Part rendering. |
| `chat.part.renderer.*` | message part renderers | renderer | chat, tools | Text/tool/reasoning/subtask. |
| `chat.tool.renderer.*` | `ToolPart.tsx`, render utils | renderer | tools | Supports exact and wildcard tool names. |
| `chat.tool.icon.*` | tool presentation | renderer/metadata | tools | Metadata registry. |
| `chat.tool.classifier` | tool presentation/classification | renderer/metadata | tools | Expandable/static/standalone policy. |
| `markdown.renderer` | Markdown renderer | renderer, replace | markdown | Heavy lazy boundary. |
| `settings.pages` | `SettingsView.tsx`, metadata | surface | settings | Replace closed settings slugs. |
| `settings.nav.groups` | settings nav | fill | settings | Runtime filtering needed. |
| `settings.sections` | settings page sections | fill | feature plugins | Page/item-level contributions. |
| `command-palette.actions` | `CommandPalette.tsx` | command | command-palette | Needs lazy search providers. |
| `command-palette.groups` | `CommandPalette.tsx` | fill/command | command-palette | Stable grouping. |
| `commands.global` | `useMenuActions`, palette | command | command-palette | Shared with native menu. |
| `files.tree.row` | file tree components | renderer, command | files | Runtime FS differences. |
| `files.editor.toolbar` | `FilesView.tsx` | fill, command | files | Dirty state and file ops. |
| `files.editor.renderer.*` | file editor modes | renderer | files | Text/markdown/json/html/etc. |
| `diff.viewer` | `DiffView`, `PierreDiffViewer` | renderer, replace | diff | Comments/scroll sensitive. |
| `git.view.sections` | `GitView`, `views/git/*` | fill, command, renderer | git, github | Large async flows. |
| `terminal.surface` | `TerminalView`, terminal components | surface, replace | terminal | Main tab and bottom dock placement. |
| `mini-chat.layout` | mini-chat app | surface, fill, wrap | mini-chat | Electron-only. |
| `vscode.agent-manager.view` | VS Code agent panel | surface, fill | agents | VS Code-only panel root. |

## Server Extension Inventory

Server extension IDs should also be product/capability IDs.

| Stable ID | Current implementation | Contribution types | Notes |
|---|---|---|---|
| `server.bootstrap` | `index.js`, `bootstrap-runtime.js` | lifecycle, middleware | Core host only. |
| `server.security` | request security/auth gates | middleware, auth-provider | Core host only initially. |
| `routes.system` | `core-routes.js` | route | `/health`, shutdown/info/free-port. |
| `routes.openchamber` | `openchamber-routes.js` | route | Models/update/system app routes. |
| `settings.schema.core` | settings helpers/runtime | settings-schema | Base settings sanitizer. |
| `auth.ui-password` | `ui-auth.js` | auth-provider | Built-in auth provider. |
| `auth.passkey` | passkey routes | auth-provider | Built-in auth provider. |
| `auth.tunnel-session` | tunnel auth | auth-provider, middleware | Tunnel-specific. |
| `opencode.lifecycle` | lifecycle runtime | lifecycle, runtime-service | Start/restart/readiness. |
| `opencode.proxy` | proxy/runtime | route, middleware, event | `/api/*` ordering sensitive. |
| `opencode.config-entities` | config entity routes | route, settings-schema | Agents, commands, MCP. |
| `sessions.runtime` | session runtime/watcher | runtime-service, event | Hot event path. |
| `event-stream.global` | event stream hub/ws | event, runtime-service | SSE/WS. |
| `notifications.core` | notification routes/runtime | route, event, runtime-service | Session awareness. |
| `notifications.push` | push runtime | route, runtime-service | Web push. |
| `tts.openai-compatible` | TTS routes/runtime | route, model-policy | TTS/STT/text. |
| `terminal.pty` | terminal runtime | route, runtime-service, lifecycle | PTY/process spawning. |
| `git.core` | Git routes/service | route, runtime-service | Git binary/process access. |
| `github.oauth` | GitHub routes | auth-provider, route | Device flow/account. |
| `github.prs` | GitHub PR routes | route, runtime-service | PR workflows. |
| `github.issues` | GitHub issue routes | route, runtime-service | Issue/pull context. |
| `fs.core` | FS routes/search | route, runtime-service | File ops/process search. |
| `preview.proxy` | preview proxy runtime | route, middleware, runtime-service | SSRF-sensitive. |
| `quota.registry` | quota routes/providers | route, runtime-service | Provider registry. |
| `skills.config` | skill routes | route, settings-schema | Skill CRUD/supporting files. |
| `skills.catalog.repo` | skills catalog | runtime-service | Repo scan/install. |
| `scheduled-tasks.runtime` | scheduled tasks runtime | lifecycle, runtime-service, event | Timers/background sessions. |
| `scheduled-tasks.routes` | scheduled task routes | route | CRUD/run/status. |
| `tunnels.registry` | tunnel registry | runtime-service | Provider registry. |
| `tunnels.provider.cloudflare` | Cloudflare provider | runtime-service | Built-in provider. |
| `magic-prompts` | magic prompt routes | route, settings-schema | Feature routes. |
| `session-folders` | session folder routes | route, settings-schema | Feature routes. |

## Server Route And Middleware Phases

Server plugin ordering must be explicit.

Recommended phases:

1. `beforeExpress`
2. `afterExpress`
3. `earlyMiddleware`
4. `bodyParser`
5. `preAuthPublicRoutes`
6. `authRoutes`
7. `authGate`
8. `postAuthFeatureRoutes`
9. `preOpenCodeProxy`
10. `openCodeProxy`
11. `postProxyRoutes`
12. `staticAssets`
13. `spaFallback`
14. `beforeListen`
15. `afterListen`
16. `beforeShutdown`
17. `afterShutdown`

Ordering rules:

- Compression must not buffer SSE routes.
- Raw body routes such as STT upload must not be broken by broad JSON parsing.
- Auth routes must be registered before the `/api` auth gate.
- Feature routes must register before the generic OpenCode proxy.
- Static SPA fallback must be last.
- WebSocket handlers must independently filter paths and verify auth/origin.
- Preview proxy WS handling must never steal terminal/message WS upgrades.

## Runtime Target Model

Suggested targets:

```ts
type PluginTarget = 'ui' | 'server' | 'vscode-extension-host' | 'electron-main';
```

### UI Target

Runs inside the shared React UI. Available in web, Electron desktop, and VS Code webview with runtime-specific limitations.

Safe APIs:

- UI registry.
- Commands registry.
- Settings registry.
- Plugin storage.
- Capability-scoped host facades for files, Git, models, tools, notifications, etc.

Unsafe APIs that should not be exposed directly:

- Raw Electron preload APIs.
- Raw `acquireVsCodeApi`.
- Raw `RuntimeAPIs` object.
- Raw Zustand stores.

### Server Target

Runs in the OpenChamber web server process. Available in web and Electron desktop. In VS Code, availability depends on whether the VS Code runtime is connected to a server that has the plugin installed.

Good for:

- API routes.
- OpenCode integrations.
- filesystem-heavy work.
- Git/network logic.
- long-lived background services.
- plugin asset serving.

### VS Code Extension Host Target

Runs only inside the VS Code extension host. This should be a later explicit target, not pretended to be portable.

Good for:

- VS Code commands.
- workspace APIs.
- editor integration.
- generating webview-safe asset URIs.

### Electron Main Target

Runs only in Electron main. This is high risk and should not be part of the first plugin phases.

Good for:

- native dialogs.
- OS integrations.
- window management.
- updater/native notification integration.

## Build-Time And Runtime Loading Strategy

### Phase 1: Built-In Plugins

Internal plugins are statically imported and registered by the host. No external loading.

### Phase 2: Build-Time Bundled External Plugins

External plugin packages are configured before build. A generated file imports them explicitly.

```ts
import ssoPlugin from '@company/openchamber-sso';
import jiraPlugin from '@company/openchamber-jira';

export const bundledPlugins = [ssoPlugin, jiraPlugin];
```

Benefits:

- Works with Vite.
- Works with Electron packaging.
- Works with VS Code webview build constraints if bundled in the webview bundle.
- Avoids duplicate React if peer dependencies are managed.
- Avoids runtime remote code security problems.

### Phase 3: Runtime Server Plugins

Server discovers allowlisted local ESM modules and loads them into the server process.

Constraints:

- Explicit allowlist.
- Capability declaration.
- Error isolation.
- Deterministic shutdown.
- No public unauthenticated routes unless explicitly allowed.

### Phase 4: Runtime UI Plugins

This is a later, high-risk phase.

Constraints:

- Signed or verified plugin bundles.
- CSP changes, especially in VS Code.
- Stable asset serving.
- Host-provided React externals.
- Capability-scoped bridge.
- Service worker cache/versioning rules.

## Host Facades

Plugins should use host facades instead of internal stores/modules.

Priority facades:

| Facade | Purpose |
|---|---|
| `host.storage` | Namespaced global/workspace plugin storage with schema/version/quota. |
| `host.settings` | Settings pages, sections, schemas, open page actions. |
| `host.commandPalette` | Command palette actions/groups/search providers. |
| `host.commands` | Global commands, native menu actions, slash command metadata. |
| `host.tools` | Tool renderers, icons, classifiers, side-effect hints. |
| `host.models` | Model filters, decorations, ranking, metadata. |
| `host.agents` | Agent listing/templates/validation through stable API. |
| `host.mcp` | MCP config/runtime/auth through stable API. |
| `host.skills` | Skills installed/catalog/installer APIs. |
| `host.features` | Feature flags and enablement state. |
| `host.directory` | Current directory/project context without closure-caching. |
| `host.runtime` | Runtime descriptor and capability checks. |

## Storage And Persistence Rules

Plugins must not write arbitrary `localStorage` keys or extend broad core stores directly.

Recommended storage namespaces:

```text
openchamber:plugins:<pluginId>:global:v<schemaVersion>
openchamber:plugins:<pluginId>:workspace:<workspaceId>:v<schemaVersion>
openchamber:plugins:index
```

Rules:

- Every plugin storage area has a schema version.
- Migrations are explicit.
- Byte and count quotas are enforced.
- Sensitive data is redacted/encrypted or routed through host credential APIs.
- Plugin uninstall can clean namespaced data.
- Persisted references to plugin IDs, surface IDs, action IDs, or model IDs must tolerate missing plugins.

## Built-In Plugin Boundaries

Recommended first-party plugin boundaries:

| Built-in plugin | Owns |
|---|---|
| `openchamber.plugin.core-workbench` | Layout, shell, slots, core surfaces, providers, runtime effects. |
| `openchamber.plugin.chat` | Chat surface, input, timeline, message base renderers. |
| `openchamber.plugin.sessions` | Session sidebar, session rows, folders, activity sections. |
| `openchamber.plugin.projects` | Projects, directories, worktrees, remote instances. |
| `openchamber.plugin.command-palette` | Global command palette, global commands, shortcut integration. |
| `openchamber.plugin.settings` | Settings shell, settings registry, built-in settings pages. |
| `openchamber.plugin.tools` | Tool renderers, classifiers, icons, side-effect hints. |
| `openchamber.plugin.models` | Model picker, model preferences, model policy contributions. |
| `openchamber.plugin.files` | Files view, file tree, editor renderers, FS APIs. |
| `openchamber.plugin.diff` | Diff viewer, inline comments, diff renderers. |
| `openchamber.plugin.git` | Git UI, Git routes, identities, commit/sync/history/conflicts. |
| `openchamber.plugin.github` | GitHub auth, PRs, issues, pickers, GitHub UI affordances. |
| `openchamber.plugin.terminal` | Terminal routes, terminal main surface, bottom dock surface. |
| `openchamber.plugin.notifications` | Notification settings, push, session attention, templates. |
| `openchamber.plugin.voice` | Voice provider, TTS/STT settings, browser voice UI. |
| `openchamber.plugin.mcp` | MCP config/runtime/auth/settings/OAuth callback. |
| `openchamber.plugin.skills` | Skills config, installed skills, catalog, skill autocomplete. |
| `openchamber.plugin.agents` | Agents settings, agent groups, agent manager. |
| `openchamber.plugin.commands` | Slash commands config and command autocomplete. |
| `openchamber.plugin.scheduled-tasks` | Scheduled task runtime, settings, dialogs, event fanout. |
| `openchamber.plugin.preview` | Preview proxy, preview/browser surfaces. |
| `openchamber.plugin.tunnels` | Tunnel registry, Cloudflare provider, tunnel settings. |
| `openchamber.plugin.mini-chat` | Electron mini-chat root and mini-chat layout. |
| `openchamber.plugin.vscode-agent-manager` | VS Code-only agent manager panel. |

## Security Rules

- Plugins cannot weaken host validation.
- UI plugins cannot be the only enforcement point for policy.
- Server route plugins default to protected routes.
- Public routes require explicit capability and manifest declaration.
- WebSocket plugins must independently verify auth and origin.
- File writes, Git writes, terminal/process execution, preview proxying, and VS Code commands are high-risk capabilities.
- OpenCode auth headers and UI session credentials must never leak to plugin-controlled external origins.
- Plugin logs must redact secrets.
- SSO/auth provider plugins must use host-owned session storage and validation.

## Performance Rules

- Plugins must not subscribe to broad hot stores.
- Hot-path slots require memoized child isolation.
- Tool renderers must avoid broad store reads and per-token expensive work.
- Command palette search providers must be lazy, cancellable, debounced, and result-limited.
- Model filtering/ranking must operate on indexed/precomputed model lists.
- Event hooks on SSE/message streams must be gated and non-blocking.
- Registries must preserve references for unchanged contribution lists.

## Diagnostics

The host should expose plugin diagnostics in UI and server APIs.

Recommended diagnostics data:

- Plugin ID, name, version, source, targets.
- Enabled/disabled state.
- Granted and denied capabilities.
- Registered contributions by type.
- Unsupported runtime targets.
- Setup errors and contribution errors.
- Replacement conflicts.
- Route/middleware phase placement.
- Storage usage.

Suggested endpoints:

- `GET /api/plugins`
- `GET /api/plugins/:pluginId`
- `GET /api/features`

## Architecture Decision Summary

- Use generic extension primitives: `fill`, `surface`, `replace`, `wrap`, `renderer`, `command`, `route`, `middleware`, `auth-provider`, `lifecycle`.
- Treat internal features as built-in plugins first.
- Keep host core responsible for layout regions, security, lifecycle, runtime bridges, storage, and diagnostics.
- Add external plugin support after internal built-ins prove the API.
- Prefer build-time bundled UI plugins before runtime UI plugins.
- Make server plugin phases explicit before accepting third-party server extensions.
