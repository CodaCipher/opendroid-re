# React Renderer Core: GUI Architecture Notes

## Overview

The main renderer bundle (`renderer-main.js`, large renderer bundle) is a Vite-bundled React 19.2.3 SPA using **TanStack Router** (file-based routing API compiled to route tree) with **hash-based history**. The app entry is the `ivo()` component, which creates a desktop-specific auth provider wrapping `window.electronAPI` and passes the router to the `vBs` (AuthProvider) component. The component hierarchy is: Root ErrorBoundary → ViewportSizeProvider → AuthGate → OrgGate → Layout (split-pane via Allotment) → RouterOutlet. The layout uses the **Allotment** library (resizable split pane) dividing the screen into a sidebar and main content area. Routing uses TanStack Router's `createRootRoute`/`La()` API with a deeply nested route tree covering sessions, settings, analytics, wiki, automations, and onboarding.

## File Map

| File | Size | Role |
|------|------|------|
| `renderer/main_window/assets/renderer-main.js` | ~8.1M bytes (large renderer bundle) | Main renderer bundle: React 19 app, all UI code |
| `renderer/main_window/index.html` | 10.5KB | Entry HTML, mounts into `<div id="root">` |
| `renderer/main_window/assets/theme.css` | 29KB | Full CSS theme |
| `build/preload.js` | small preload bridge | IPC bridge (`window.electronAPI`) |

## Architecture

### React Root Creation

```js
hqr.createRoot(document.getElementById("root")).render(m.jsx(ivo, {}));
```

The `hqr` variable is `ReactDOM`. The root mounts into `<div id="root">` (confirmed in `index.html`). The top-level app component is `ivo()`.

### Component Hierarchy (Provider Tree)

```
createRoot(#root)
└── ivo()                         : Desktop auth bootstrap
    └── vBs                        : AuthProvider
        ├── authContext = {
        │     isAuthenticated, user, isLoading,
        │     signIn, signOut, getAccessToken,
        │     switchToOrganization, cancelSignIn
        │   }
        └── Iwi (RouterProvider)   : TanStack Router
            └── Root ErrorBoundary (gvr, variant="root")
                └── jsi()                         : ViewportSizeProvider
                    │   Context: qPe (Compact|Medium|Wide|Expanded)
                    │   Uses matchMedia breakpoints
                    └── J0o()                     : Main layout wrapper
                        ├── Z0o()                  : DebugMode flag bar (conditional)
                        └── Y0o()                   : AuthGate
                            ├── Loading spinner while auth loading
                            ├── If !isAuthenticated → pass through (login page)
                            └── If authenticated → K0o() : OrgDataGate
                                ├── Loading spinner
                                └── $0o()               : OnboardingGate
                                    ├── Redirect to onboarding if !onboarded
                                    ├── If onboarded → qUs (SubscriptionGate)
                                    └── NBs (NotificationsProvider)
                                        └── wZs (ShortcutsProvider)
                                            └── children (route outlet)
```

### Top-Level Layout (Split Pane: Allotment)

The main layout component `a0o()` (the "allotment" route) uses **Allotment** (`B9` component) to create a resizable split pane:

```
te { direction: column, height: 100%, width: 100% }
├── Ce { flex: 1, overflow: hidden, position: relative }
│   └── [Title bar / navigation header with debug flags]
└── B9 (Allotment) { onChange: persist sizes }
    ├── B9.Pane { minSize: 0|400 } : Sidebar
    │   ├── KDn(): Sidebar content (sessions, settings, wiki, analytics)
    │   └── Sidebar modes: "settings" | "analytics" | "wiki"
    └── B9.Pane { minSize: 0|400 } : Main content
        └── q0e(): RouterOutlet (renders matched route component)
```

The sidebar uses animated pan transitions (`sidebar-pan-exit-left`, `sidebar-pan-enter-right`) when switching between modes. The sidebar can be toggled with keyboard shortcut **Ctrl+B**.

### Keyboard Shortcuts (Global)

| Shortcut | Action |
|----------|--------|
| Ctrl+N | Navigate to sessions (new session) |
| Ctrl+B | Toggle sidebar |
| Ctrl+J | Toggle terminal panel |
| Ctrl+O | Toggle mission panel |
| Ctrl+K | Open command palette / search |

### Route Structure

The route tree is built via `y0o({layout, homePage})` using TanStack Router's nested route API:

```
RootRoute (layout: evo, homePage: _0o)
├── _authenticated (beforeLoad: check auth)
│   ├── allotment (component: a0o: split pane layout)
│   │   ├── /sessions                   : Session list (LXs)
│   │   ├── /sessions/$sessionId        : Session detail (uXs)
│   │   ├── /sessions/new               : Redirect to sessions list
│   │   ├── /settings                   : Settings (Rgo)
│   │   │   ├── /settings/              : General settings (ngo)
│   │   │   ├── /settings/organization  : Org settings (mgo)
│   │   │   ├── /settings/api-keys      : API keys (UXs)
│   │   │   ├── /settings/billing       : Billing (iZs)
│   │   │   ├── /settings/usage         : Usage stats (Hgo)
│   │   │   ├── /settings/cloud-templates: Cloud templates (IZs)
│   │   │   ├── /settings/droid-computers: Computer list (zho)
│   │   │   ├── /settings/droid-computers/$computerId: Computer detail (Cho)
│   │   │   ├── /settings/enterprise-controls: Enterprise (Vpo)
│   │   │   ├── /settings/support       : Support (Ogo)
│   │   │   └── /settings/voucher       : Voucher (Vgo)
│   │   ├── /analytics                  : Analytics (szs)
│   │   │   ├── /analytics/             : Overview (hzs)
│   │   │   ├── /analytics/me           : Personal stats (Azs)
│   │   │   ├── /analytics/tokens       : Token usage (gHs)
│   │   │   ├── /analytics/tools        : Tool usage (NHs)
│   │   │   ├── /analytics/activity     : Activity log (JBs)
│   │   │   ├── /analytics/productivity : Productivity (Dzs)
│   │   │   ├── /analytics/users        : User analytics (QHs)
│   │   │   ├── /analytics/readiness    : Agent readiness (SFs)
│   │   │   └── /analytics/readiness/$repoId: Repo readiness (Jzs)
│   │   ├── /automations                : Automations list (dVs)
│   │   ├── /wiki                       : Wiki index (vmo)
│   │   └── /wiki/$wikiRunId            : Wiki page view (L1o)
│   ├── /onboarding                     : Onboarding wizard (sQs)
│   ├── /onboarding/finish              : Onboarding completion (aQs)
│   ├── /settings/integrations/$integration/start: Integration start (sgo)
│   ├── /settings/integrations/$integration/callback: Integration callback (rgo)
│   └── /new-session                    : Redirect to sessions with prefill prompt
├── / (home)                            : Login/landing (e = _0o, signIn page)
├── /health                             : Health check (bVs)
├── /debug                              : Debug page (fVs)
├── /callback                           : Auth callback (SBs)
├── /login                              : Login page (Wgo)
├── /logout                             : Logout page (qgo)
├── /close-window                       : Close window (Qgo)
├── /mcp-oauth-callback                 : MCP OAuth (mcpOAuthCallback: $go)
├── /e2e-login                          : E2E test login (pVs)
├── /referral                           : Referral page (uQs)
├── /settings/integrations-error        : Integration error (igo)
└── [Redirect routes]: settings/integrations→settings/organization,
    settings/security→settings/enterpriseControls, settings/session→cloudTemplates,
    agentReadiness→analytics/readiness, droids→home,
    agentReadinessReports→analytics/readiness, cliOnboarding→onboarding?origin=cli,
    voucher→settings/voucher
```

### Router Configuration

```js
const rvo = nvo({ historyType: "hash" });
// nvo creates the router:
function nvo({ historyType: t } = {}) {
  let e;
  t === "hash" && (e = zSi());  // createHashHistory
  const n = Cwi({               // createRouter
    routeTree: tvo,
    history: e,
    scrollRestoration: true,
    scrollToTopSelectors: [yBs],
    context: { auth: undefined }
  });
  return window.OpenDroidRouter = n, n;
}
```

Key details:
- **Hash-based routing** (`historyType: "hash"`): uses `createHashHistory()` (`zSi`)
- Router instance exposed globally as `window.OpenDroidRouter`
- `scrollRestoration: true`: TanStack Router handles scroll position
- Route context includes `auth` (set by the auth provider)

### Context/Provider Catalog

| Context Variable | Provider Component | Purpose | Value Type |
|-----------------|-------------------|---------|------------|
| `vBs` (auth) | `ivo()` → `vBs` | Desktop auth state + methods | `{isAuthenticated, user, isLoading, signIn, signOut, getAccessToken, switchToOrganization, cancelSignIn}` |
| `Cpt` (router) | TanStack Router | Router state, location, navigation | Internal TSR state |
| `qPe` (viewport) | `jsi()` | Responsive breakpoint | `"compact" \| "medium" \| "wide" \| "expanded"` |
| `n5t` (debug) | Feature flags context | Debug mode toggle, feature flags | `{debugMode, flags, toggleFlag, toggleExclusive}` |
| `fkt` (settings) | Settings resolution chain | Feature flag resolution | Resolution chain for debugging |
| `AUe` (match ID) | TanStack Router | Current route match ID | Match ID string |
| `NBs` (notifications) | Notification provider | Notification state | Notification state |
| `wZs` (shortcuts) | Keyboard shortcuts provider | Global hotkeys | Shortcut registrations |
| `qUs` (subscription) | Subscription/onboarded gate | Onboarding + subscription state | Subscription data |
| `LUn` (styled-components) | Styled-components StyleSheet | CSS-in-JS theming | StyleSheet instance |
| `UUn` (SC theme) | Styled-components theme | Theme override | Theme object |

### Auth Integration (Desktop-Specific)

The `ivo()` component is the desktop-specific auth bootstrap:

```text
ivo():
  create desktop auth state
  on mount:
    require window.electronAPI
    read auth data once
    subscribe to auth success events
    subscribe to navigation/deeplink events
    cleanup subscriptions on unmount

  provide auth context methods:
    signIn, signOut, getAccessToken, switchToOrganization, cancelSignIn
```

### Error Boundaries

Two levels of error boundary using `gvr()` wrapper around `X0o` (React class component):
- **Root level** (`variant: "root"`): wraps the entire app
- **Route level** (`variant: "route"`): wraps individual route components

Error boundary reports to ErrTracker via `oee()` and increments `WEB_ERROR_BOUNDARY_TRIGGERED_COUNT` metric.

### Command Palette

A global command palette (`jbo` component) provides:
- Session search (local + cloud results)
- Recent commands
- Recent sessions
- Sub-pages navigation

Opened with **Ctrl+K**.

### Route Constants (Cr object)

Key route path constants visible in the route tree:
- `Cr.home`: Login/landing
- `Cr.onboarding`: Onboarding wizard
- `Cr.onboardingFinish`: Onboarding completion
- `Cr.sessions.all`: Session list
- `Cr.sessions.new`: New session
- `Cr.settings.general`: Settings root
- `Cr.settings.organization`: Organization settings
- `Cr.settings.integrations`: Integrations (redirects)
- `Cr.settings.integrationsError`: Integration errors
- `Cr.settings.enterpriseControls`: Enterprise controls
- `Cr.settings.cloudTemplates`: Cloud templates
- `Cr.settings.voucher`: Voucher
- `Cr.settings.agentReadiness`: Agent readiness (redirects)
- `Cr.analytics.agentReadiness`: Agent readiness analytics
- `Cr.automations.all`: Automations list
- `Cr.wiki`: Wiki index
- `Cr.droids`: Droids (redirects to home)
- `Cr.login`: Login
- `Cr.logout`: Logout
- `Cr.callback`: Auth callback
- `Cr.debug`: Debug
- `Cr.health`: Health check
- `Cr.closeWindow`: Close window
- `Cr.mcpOAuthCallback`: MCP OAuth callback
- `Cr.e2eLogin`: E2E test login
- `Cr.referral`: Referral
- `Cr.newSession`: New session (redirects)
- `Cr.cliOnboarding`: CLI onboarding (redirects)

## Key Findings

### Finding 1: TanStack Router, Not React Router
The application uses **TanStack Router** (v1.x file-based routing API), not React Router. The routing API uses `createRootRoute` (`pwi()`), `createRoute` (`La()`), and `createRouter` (`Cwi()`). Routes are defined via `La({getParentRoute, id, path, component, beforeLoad, validateSearch})` and composed with `.addChildren()`.

### Finding 2: Hash-Based Routing for Electron
The Electron desktop app uses **hash-based routing** (`createHashHistory` / `zSi()`), which avoids issues with file:// protocol and deep linking in packaged Electron apps.

### Finding 3: Auth is Desktop-Specific via electronAPI
The `ivo()` component directly accesses `window.electronAPI` (confirmed from preload bridge: the namespace is `electronAPI`, not `api`). It listens for `auth:success` push events and `navigate` push events from the main process, enabling deeplink navigation and notification-triggered routing.

### Finding 4: Allotment Split Pane Layout
The main layout uses the **Allotment** library (`B9` component) for resizable split panes. The left pane contains a sidebar with animated mode switching (sessions, settings, analytics, wiki). The right pane contains the main route content. Pane sizes are persisted.

### Finding 5: Three-Layer Gate System
Before the main app renders, three gate components check conditions:
1. **AuthGate** (`Y0o`): waits for auth loading, passes through if not authenticated (login page)
2. **OrgDataGate** (`K0o`): fetches organization data
3. **OnboardingGate** (`$0o`): redirects to onboarding if org not onboarded, wraps in SubscriptionGate (`qUs`) if onboarded

### Finding 6: ViewportSizeProvider for Responsive Design
The `jsi()` component provides a responsive breakpoint context (`qPe`) using `matchMedia` queries:
- Compact, Medium, Wide, Expanded breakpoints
- Components use `j0()` hook to read current size

### Finding 7: React 19.2.3 with Compiler
The bundle includes React 19.2.3 with `__COMPILER_RUNTIME` support (React Compiler's `useMemoCache`). This enables automatic memoization via the compiler's cache system (`Ae.c(N)` calls seen throughout).

### Finding 8: ErrTracker Integration
Full ErrTracker Electron integration is present:
- ErrTracker error boundary reporting
- ErrTracker breadcrumb for debug logs
- ErrTracker IPC bridge in preload (fire-and-forget via `ipcRenderer.send`)
- Performance monitoring (CLS, LCP, TTFB, INP spans)

### Finding 9: styled-components for Design System
The UI uses **styled-components** (`Xi` / `Xi.elementName` pattern) for styled elements, with a StyleSheetManager context. Design tokens are defined as styled-components theme.

### Finding 10: Minification Impact
Minification obscures approximately 60% of component names. However, the route tree preserves meaningful path names (`Cr.sessions.all`, `Cr.settings.organization`), and the component hierarchy is reconstructable from the JSX structure and hook patterns.

## Code Examples

### React Root and App Bootstrap (end of bundle)
```js
// Line ~last section of renderer-main.js
hqr.createRoot(document.getElementById("root")).render(m.jsx(ivo, {}));
```

### Desktop Auth Provider (ivo component)
```text
ivo():
  create desktop auth state
  on mount:
    require window.electronAPI
    read auth data once
    subscribe to auth success events
    subscribe to navigation/deeplink events
    cleanup subscriptions on unmount

  provide auth context methods:
    signIn, signOut, getAccessToken, switchToOrganization, cancelSignIn
```

### Router Creation with Hash History
```js
const rvo = nvo({ historyType: "hash" });

function nvo({ historyType: t } = {}) {
  let e;
  t === "hash" && (e = zSi());  // createHashHistory()
  const n = Cwi({               // createRouter()
    routeTree: tvo,
    history: e,
    scrollRestoration: true,
    scrollToTopSelectors: [yBs],
    context: { auth: void 0 }
  });
  return window.OpenDroidRouter = n, n;
}
```

### Main Layout with Allotment Split Pane
```js
// Simplified from a0o() component
m.jsxs(B9, { onChange: Ge, children: [
  // Sidebar pane
  m.jsx(B9.Pane, { minSize: $e, children: sidebarContent }),
  // Main content pane
  m.jsx(B9.Pane, { minSize: $e2, children: m.jsx(q0e, {}) })  // RouterOutlet
] })
```

### Route Tree Construction (partial)
```js
function y0o({ layout: t, homePage: e }) {
  const n = pwi()({ component: t, notFoundComponent: F9n });
  const i = La({ getParentRoute: () => n, id: "_authenticated",
    beforeLoad: ({ context: Ue, location: $e }) => {
      if (!Ue.auth.isAuthenticated) throw G3({ to: Cr.home });
    }
  });
  const o = La({ getParentRoute: () => i, id: "allotment", component: a0o });
  const l = La({ getParentRoute: () => o, path: Cr.sessions.all, component: LXs });
  // ... more routes ...
  return n.addChildren([i.addChildren([o.addChildren([...]), ...]), ...]);
}
```

## Theme / Style Tokens

Not the primary focus of this feature (see `gui-theme-css.md`). However, key observations:
- Theme switching via `data-theme="dark"` attribute on `<html>` element
- Debug flag style overrides can modify `--surface-1` CSS variable for different background colors
- styled-components used for component-level styling

## IPC Channels

### IPC Channels Used by Renderer Core (Auth + Navigation)

| Channel | Direction | Usage in Renderer Core |
|---------|-----------|----------------------|
| `auth:getAuthData` | R→M→R (invoke) | Called in `ivo()` on init and after `auth:success` push |
| `auth:signIn` | R→M→R (invoke) | Called from auth context `signIn()` method |
| `auth:signOut` | R→M→R (invoke) | Called from auth context `signOut()` method |
| `auth:success` | M→R (push via on) | Listener in `ivo()`: refreshes auth data |
| `auth:switchToOrganization` | R→M→R (invoke) | Called from auth context `switchToOrganization()` |
| `navigate` | M→R (push via on) | Listener in `ivo()`: sets `window.location.hash` |

### Other IPC channels are used by specific route components (see `gui-renderer-ipc-client.md`)

## Integration Points

### Cross-System Dependencies

1. **Preload IPC Bridge** (`gui-preload-bridge.md`): The renderer accesses `window.electronAPI` for auth and navigation. All 21 methods are available; the core renderer uses 6 (5 auth + 1 navigation).

2. **Main Process Auth Handlers** (`gui-main-process-ipc.md`): The main process must handle `auth:signIn`, `auth:signOut`, `auth:getAuthData`, `auth:switchToOrganization` via `ipcMain.handle`, and push `auth:success` and `navigate` events.

3. **Main Process Daemon** (`gui-main-process-daemon.md`): The `navigate` push event likely originates from daemon WebSocket messages that trigger the main process to instruct the renderer to navigate (deeplinks, notification clicks).

4. **HTML Entry Point** (`gui-html-entry.md`): The `<div id="root">` mount point, theme detection script, and CSP are all prerequisites for the renderer.

5. **ErrTracker IPC** (`__ERRTRACKER_IPC__`): The renderer sends error/performance data to the main process via the ErrTracker IPC bridge for relay to ErrTracker.

6. **Secondary Chunks** (`renderer-chunk-a.js`, `renderer-chunk-b.js`): Lazy-loaded route components referenced via dynamic imports from the main bundle.

7. **CSS Theme** (`theme.css`): The `data-theme` attribute used by the theme detection script directly controls CSS variable resolution.

### Cross-Mission References

- **Auth flow** → CLI/TUI may share auth tokens (Terminal UI section, Tool & Agent section scope)
- **Daemon WebSocket** → connects to opendroid.exe (Infrastructure section scope)
- **SandboxSvc sandbox** → sandbox execution environment (Infrastructure section scope)
- **IdProvider authentication** → auth provider (Infrastructure section scope)

## Implementation Notes

### Porting the Renderer Core

1. **Router**: Replace TanStack Router with your preferred router (React Router, Next.js App Router, etc.). The route tree structure is clean and can be directly mapped. Keep hash-based routing for Electron compatibility or switch to memory history if preferred.

2. **Auth Provider**: The desktop auth pattern (`ivo()` component) is Electron-specific. For OpenDroid:
   - If using Electron: keep the `window.electronAPI` pattern
   - If using Tauri: replace with `window.__TAURI_INVOKE__` or Tauri's invoke API
   - If web-only: replace with a standard OAuth flow

3. **Split Pane Layout**: The Allotment library can be replaced with any resizable split pane component (e.g., `react-split`, `allotment` itself, or a custom CSS grid solution). The sidebar + main content pattern is universal.

4. **Gate System**: The three-layer gate (Auth → OrgData → Onboarding) is a clean pattern. Port this directly: it provides a good separation of concerns for initial loading states.

5. **Context Architecture**: The context/provider pattern (auth, viewport, debug, notifications, shortcuts) is framework-agnostic. Consider using Zustand or Jotai for state management instead of plain React context for better performance.

6. **Route Constants**: The `Cr` object pattern for route paths is excellent for type-safe routing. Port this as a constants module.

7. **Error Boundaries**: The two-level error boundary (root + route) with ErrTracker reporting is a best practice. Port with your preferred error reporting service.

8. **Global Shortcuts**: The `bd` component for keyboard shortcuts is reusable. Map to your preferred keyboard shortcut library.

9. **ViewportSizeProvider**: The responsive breakpoint context using `matchMedia` is portable as-is.

10. **Minification Note**: When building OpenDroid, consider keeping component names in development builds for easier debugging. The OpenDroid production build heavily minifies, making architecture research necessary.

## Module Reference

| File | Offset Range | Symbol / Function | Notes |
|------|--------------|-------------------|-------|
| `renderer-main.js` | ErrTracker init section | React 19.2.3 internals | `Ac.createContext`, `Ac.useState`, etc. |
| `renderer-main.js` | ~Line 49 | `createRoot` + DOM operations | `hqr.createRoot` |
| `renderer-main.js` | ~Line 3138 | TanStack Router internals | `createRootRoute` (`pwi`), `createRoute` (`La`), `createRouter` (`Cwi`) |
| `renderer-main.js` | ~Line 2382 | ViewportSizeProvider (`jsi`) | `qPe` context, matchMedia breakpoints |
| `renderer-main.js` | End section | `ivo()`: Desktop auth bootstrap | `window.electronAPI` auth + navigate |
| `renderer-main.js` | End section | `nvo()`: Router creation | Hash history, route tree, OpenDroidRouter global |
| `renderer-main.js` | End section | `y0o()`: Route tree builder | All route definitions with `La()` |
| `renderer-main.js` | End section | `a0o()`: Allotment layout | Split pane, sidebar, main content |
| `renderer-main.js` | End section | `Y0o()`: AuthGate | Loading, unauthenticated pass-through |
| `renderer-main.js` | End section | `K0o()`: OrgDataGate | Fetches org data |
| `renderer-main.js` | End section | `$0o()`: OnboardingGate | Redirects to onboarding |
| `renderer-main.js` | End section | `X0o`: ErrorBoundary class | ErrTracker reporting, path logging |
| `renderer-main.js` | End section | `G0o()`: DebugMode flags bar | Feature flag toggle UI |
| `renderer-main.js` | End section | `KDn()`: Sidebar content | Sessions, settings, analytics, wiki modes |
| `renderer-main.js` | End section | `hqr.createRoot(...)` | Final React mount |
