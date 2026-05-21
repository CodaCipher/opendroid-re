# Onboarding Wizard & Tutorial Modal: GUI Architecture Notes

## Overview

The OpenDroid Electron GUI implements a two-layer onboarding system: (1) a **4-step wizard state machine** (`rQs`) that collects organization name, handles payment method setup, and completes backend registration, and (2) a **7-slide tutorial modal** (`t0o`) that auto-plays theme-aware MP4 walkthrough videos. Onboarding completion is gated by an `onboarded` boolean on the organization object fetched from `/api/auth/me`. Un-onboarded users are redirected to `/onboarding`; onboarded users land on `/sessions`. The tutorial modal is triggered independently in the main app layout based on a `tutorialModalCompleted` flag.

## File Map

| File | Size | Role |
|------|------|------|
| `renderer-main.js` | ~large renderer bundle | Main renderer bundle: contains onboarding state machine, tutorial modal, auth gate, theme system |
| `step1-dark.mp4` | ~3.4 MB | Step 1 video (dark theme) |
| `step1-light.mp4` | ~3.4 MB | Step 1 video (light theme) |
| `step2.mp4` | ~2.1 MB | Step 2 video (theme-agnostic) |
| `step3.mp4` | ~1.8 MB | Step 3 video (theme-agnostic) |
| `step4.mp4` | ~2.5 MB | Step 4 video (theme-agnostic) |
| `theme.css` | 29 KB | Theme CSS variables: `--surface-*`, `--text-*`, `--border-*` |

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Auth Gate ($0o)                                                │
│  ├─ Fetches user + organization from /api/auth/me (z0 hook)     │
│  ├─ Checks !!organization?.onboarded                            │
│  ├─ Redirects to /onboarding if NOT onboarded                   │
│  └─ Redirects to /sessions if onboarded                         │
├─────────────────────────────────────────────────────────────────┤
│  Onboarding Route (/onboarding)                                 │
│  ├─ sQs: Wrapper: validates promo, determines auth method      │
│  │   └─ rQs: 4-step state machine                               │
│  │       ├─ SIGNUP_WARNING → NAME → PAYMENT → FINAL             │
│  │       ├─ Calls POST /api/organization/create (tQs)           │
│  │       ├─ Calls POST /api/organization/onboarding/complete    │
│  │       └─ Navigates to /onboarding/finish                     │
│  └─ aQs: Finish page: VS Code, JetBrains, CLI, desktop links   │
├─────────────────────────────────────────────────────────────────┤
│  Tutorial Modal (t0o): triggered in main layout (a0o)          │
│  ├─ 7-slide carousel with useState(0) index                     │
│  ├─ Slides 2-5 use <video autoPlay loop muted playsInline>      │
│  ├─ Step-1 video switches dark/light via yNs theme map          │
│  └─ Completion → setState({tutorialModalCompleted: true})       │
└─────────────────────────────────────────────────────────────────┘
```

## Key Findings

### 1. Onboarding State Machine (`rQs`)

**Location:** `renderer-main.js`, lines ~9967–10050  
**Steps:** `SIGNUP_WARNING` → `NAME` → `PAYMENT` → `FINAL`

The state machine uses React `useState` for the current step:
```js
[K, Z] = B.useState(C ? M ? $_.FINAL : N||t&&!f ? $_.PAYMENT : $_.FINAL
                    : t||M ? $_.NAME : $_.SIGNUP_WARNING)
```

**Step logic:**
- If organization already exists (`C`):
  - Skip payment (`M`) → `FINAL`
  - Needs payment (`N`) or verified method + not grant → `PAYMENT`
  - Otherwise → `FINAL`
- If no organization:
  - Verified method or skip payment → `NAME`
  - Otherwise → `SIGNUP_WARNING`

**Step definitions (from `$_` enum at ~line 9966):**
| Enum Key | String Value | Used in rQs |
|----------|-------------|-------------|
| `SIGNUP` | `signup` | No |
| `SIGNUP_WARNING` | `signup-warning` | Yes (entry point for email users) |
| `NAME` | `org-name` | Yes (organization name input) |
| `PAYMENT` | `payment` | Yes (Stripe payment link) |
| `FINAL` | `finish-onboarding` | Yes (completion + API call) |
| `NEXT_STEP` | `next step` | No |
| `SELECT_USAGE` | `select-usage` | No |
| `INTEGRATIONS` | `integrations` | No |
| `INSTALL_CLI` | `install-cli` | No |
| `SETUP_IDE` | `setup-ide` | No |
| `DESKTOP_DOWNLOAD` | `desktop-download` | No |
| `WORKING_DIRECTORY` | `working-directory` | No |
| `DEFINITIONS_FACTORY` | `definitions-factory` | No |
| `DEFINITIONS_DROID` | `definitions-droid` | No |
| `BRIDGE` | `bridge` | No |

The enum defines 15 steps total, but the current `rQs` implementation only uses 4 of them. The unused steps suggest a legacy or planned expansion of the onboarding flow.

**Payment flow:**
- Uses `XFs` hook to generate a Stripe checkout URL (`returnUrl` + `cancelUrl`)
- Payment is skipped if promo code has `skipPaymentMethod: true`
- User opens Stripe in a new window via `window.open(ge, "_blank")`

**Completion flow:**
- `ye()` calls `ie.mutateAsync({promotionId, grantIntent})` → `POST /api/organization/onboarding/complete`
- Invalidates `auth/me` query cache
- Navigates to `/onboarding/finish?tier={paid|free}&origin={cli|web}`

### 2. Tutorial Modal (`t0o`)

**Location:** `renderer-main.js`, lines ~10539–10566 (end of file)  
**Type:** 7-slide carousel component

**Slide sequence:**
1. **Welcome**: static content (no video)
2. **Interacting with Droid**: video slide
3. **Capabilities**: video slide
4. **CLI**: video slide
5. **Desktop**: video slide
6. **Integrations**: video slide
7. **Feedback**: static content (no video)

**Video component (`$si`):**
```js
m.jsx($si, { autoPlay: !0, loop: !0, muted: !0, playsInline: !0, src: ... })
```

**State management:**
```js
[i, o] = B.useState(0)  // current slide index
```

**Navigation:**
- "Continue" button increments index
- On last slide, "Continue" calls `onComplete()` prop
- The component exposes `onComplete` callback to parent

### 3. Theme-Aware Video Selection (`yNs`)

**Location:** `renderer-main.js`, lines 9296–9301

**Video asset catalog:**
```js
const bNs = "" + new URL("step1-dark.mp4", import.meta.url).href
const vNs = "" + new URL("step1-light.mp4", import.meta.url).href
const jMn = "" + new URL("step2.mp4", import.meta.url).href
const BMn = "" + new URL("step3.mp4", import.meta.url).href
const UMn = "" + new URL("step4.mp4", import.meta.url).href

const yNs = {
  dark:  { step1: bNs, step2: jMn, step3: BMn, step4: UMn },
  light: { step1: vNs, step2: jMn, step3: BMn, step4: UMn }
}
```

**Key observation:** Only Step 1 has dark/light variants. Steps 2–4 are theme-agnostic. The theme is obtained via `xB()` hook (`const {theme: l} = xB()`), and the video src is selected as `yNs[l]["step" + (i+1)]` (where `i` is the slide index).

### 4. Auth Gate & Onboarding Redirect (`$0o`)

**Location:** `renderer-main.js`, lines ~10540–10560

The `$0o` component wraps the entire authenticated app and enforces onboarding state:

```js
function $0o({children:t, data:e, error:n}) {
  const u = e?.organization
  const h = !!u?.onboarded        // true = onboarding complete
  const g = zT(u || null)         // checks org validity
  const b = i.pathname === Cr.onboarding

  // Redirect logic:
  const M = n ? null
    : !u || !h && !g ? (b ? null : Cr.onboarding)
    : (y || b ? Cr.sessions.all : null)

  useEffect(() => { if (!n && M) navigate({to: M, search: i.search}) })
}
```

**Behavior:**
- If no organization OR (`!onboarded` AND `!zT(org)`) → redirect to `/onboarding`
- If already on `/onboarding` and condition met → allow (don't redirect)
- If `onboarded` → redirect to `/sessions`

This creates a hard gate: users cannot access the main app until their organization has `onboarded: true`.

### 5. Tutorial Modal Trigger & Persistence

**Location:** Main layout component (`a0o`), near end of file (~lines 10520–10538)

The tutorial modal is conditionally rendered:
```js
const st = !!(P && N)   // condition for showing modal
// ...
m.jsx(Ca, {            // Modal wrapper
  onClose: Eme.noop,   // non-dismissible
  open: st,
  variant: ls.Plain,
  children: m.jsx(t0o, {
    onComplete: () => {
      D(!1),            // close modal
      T({tutorialModalCompleted: !0})  // persist flag
    }
  })
})
```

**Persistence mechanism:** The `tutorialModalCompleted` flag is passed to a state setter `T` (likely from a `useState` or similar hook in the parent layout). The exact storage mechanism (localStorage vs. backend state) is obscured by React compiler output, but the flag name suggests it's tracked in user or app state.

The modal is **non-dismissible** (`onClose: Eme.noop`), meaning users must complete all 7 slides before accessing the app.

### 6. Theme System (`xB` hook)

**Location:** `renderer-main.js`, lines ~9966+

```js
const Xze = "opendroid-theme-preference"    // localStorage key
const z9n = "data-theme"                  // HTML attribute

function SVs() {   // System theme detector
  return window.matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light"
}

function EVs() {   // Stored theme reader
  const t = localStorage.getItem(Xze)
  return t === "light" || t === "dark" ? t : "dark"   // default dark
}

function xB() {    // Theme hook
  const [e, n] = zo(wVs)   // zustand/store hook
  const toggleTheme = () => { const S = e === "dark" ? "light" : "dark"; n(S); syt(S) }
  return { theme: e, setTheme: n, toggleTheme, systemTheme: o, hasPreference, clearPreference }
}

function syt(t) {  // Sync theme to DOM
  t === "dark" ? document.documentElement.setAttribute(z9n, "dark")
               : document.documentElement.removeAttribute(z9n)
}
```

**Default theme:** `dark` (hardcoded fallback).  
**Storage:** `localStorage` with `opendroid-theme-preference` key.  
**DOM sync:** `data-theme="dark"` attribute on `<html>` element.

### 7. Backend API Endpoints

| Function | Endpoint | Method | Purpose |
|----------|----------|--------|---------|
| `eQs()` | `/api/organization/onboarding/complete` | POST | Marks onboarding as complete |
| `tQs()` | `/api/organization/create` | POST | Creates organization with name |
| `z0()` (auth me) | `/api/auth/me` | GET | Fetches user + org + `onboarded` flag |
| `XFs` hook | `/api/payment/checkout` (inferred) | GET | Generates Stripe checkout URL |
| `iQs()` | `/api/promotions/{code}/validate` | POST | Validates promotion code |

### 8. Onboarding Finish Page (`aQs`)

**Location:** `renderer-main.js`, lines ~10050+

The finish page (`/onboarding/finish`) shows different content based on:
- `tier`: `"paid"` vs `"free"`: determines subscription status messaging
- `origin`: `"cli"` vs web: determines which install options to highlight

**Available actions:**
- VS Code extension install link (`vscode:extension/OpenDroid.opendroid-vscode-extension`)
- JetBrains plugin link (`https://plugins.jetbrains.com/plugin/opendroid`)
- CLI install commands (mac: `curl -fsSL https://app.opendroid.dev/cli | sh`, windows: `irm https://app.opendroid.dev/cli/windows | iex`)
- Desktop download link
- Billing/settings link for paid users

## Code Examples

### Video Definition Block (lines 9296–9301)
```js
const bNs=""+new URL("step1-dark.mp4",import.meta.url).href
const vNs=""+new URL("step1-light.mp4",import.meta.url).href
const jMn=""+new URL("step2.mp4",import.meta.url).href
const BMn=""+new URL("step3.mp4",import.meta.url).href
const UMn=""+new URL("step4.mp4",import.meta.url).href
const yNs={dark:{step1:bNs,step2:jMn,step3:BMn,step4:UMn},
            light:{step1:vNs,step2:jMn,step3:BMn,step4:UMn}}
```

### Onboarding State Machine Initialization (within `rQs`)
```js
[K, Z] = B.useState(
  C ? M ? $_.FINAL : N||t&&!f ? $_.PAYMENT : $_.FINAL
    : t||M ? $_.NAME : $_.SIGNUP_WARNING
)
```

### Organization Creation + Onboarding Complete
```js
we = async() => {
  const Ge = (await J.mutateAsync({name: q, signupTouch: Ve})).data?.idpOrgId
  await y({organizationId: Ge})
  window.location.reload()
}
ye = async() => {
  const Ke = (await ie.mutateAsync({promotionId: c??void 0, grantIntent: h})).data?.organization
  await o.invalidateQueries({queryKey: ["auth", "me"]})
  i({to: Cr.onboardingFinish, search: {tier: Ge?"paid":"free", origin: u}})
}
```

### Tutorial Modal Video Slide
```js
function t0o(t) {
  const [i, o] = B.useState(0)   // slide index
  const {theme: l} = xB()
  const Ce = yNs[l]["step" + (i+1)]
  // ...
  m.jsx($si, {autoPlay: !0, loop: !0, muted: !0, playsInline: !0, src: Ce})
}
```

### Auth Gate Redirect Logic (`$0o`)
```js
const u = e?.organization
const h = !!u?.onboarded
const g = zT(u || null)
const b = i.pathname === Cr.onboarding
const M = n ? null
  : !u || !h && !g ? (b ? null : Cr.onboarding)
  : (y || b ? Cr.sessions.all : null)
useEffect(() => { !n && M && navigate({to: M}) })
```

## Theme / Style Tokens

The onboarding flow uses the following CSS custom properties (from `theme.css`):

| Token | Usage |
|-------|-------|
| `--surface-1` | Modal background, card backgrounds |
| `--surface-2` | Debug/secondary panels |
| `--surface-3` | Inline code/highlight backgrounds |
| `--surface-4` | Tag/badge backgrounds |
| `--text-label` | Badge text color |
| `--border-1` | Card borders, dividers |
| `--radius-sm` | Small border radius (buttons, tags) |
| `--radius-md` | Medium border radius (panels) |
| `--font-mono` | Monospace font for commands/IDs |

Theme switching is handled by `data-theme="dark"` on the `<html>` element. The onboarding components consume theme via the `xB()` React hook.

## IPC Channels

No direct `ipcRenderer` or `window.electronAPI` calls were found within the onboarding wizard or tutorial modal components. The onboarding flow operates purely through HTTP API calls to the backend.

However, the auth system (which the onboarding gate depends on) uses:
- `window.electronAPI.auth.getAuthData()`: fetched in `ivo()` at app bootstrap
- `window.electronAPI.auth.signIn()` / `signOut()`: for auth state changes
- `window.electronAPI.auth.switchToOrganization({organizationId})`: used after org creation

These are documented in `gui-preload-bridge.md` and `gui-main-process-ipc.md` (other 04-desktop-gui reports).

## Integration Points

| System | Integration | Notes |
|--------|-------------|-------|
| **Backend API** | `POST /api/organization/create`, `POST /api/organization/onboarding/complete`, `GET /api/auth/me` | Core onboarding persistence |
| **Stripe** | Checkout URL generation via `XFs` hook | Payment collection for paid tier |
| **IdProvider** | Organization creation returns `idpOrgId` | Identity/organization management |
| **Promotions** | `/api/promotions/{code}/validate` | Promo code validation for free credits |
| **Google Analytics** | `BG("onboarding_{event}")` | Event tracking for funnel analysis |
| **Auth Context** | `_c()` hook, `vBs` provider | User state from main process via preload |
| **TanStack Router** | `Uc()` hook, `Cr.*` route constants | Navigation between onboarding steps |
| **TanStack Query** | `Vl()` query client, `invalidateQueries` | Cache invalidation after org creation |
| **Theme System** | `xB()` hook, CSS custom properties | Dark/light mode for Step 1 video |
| **Desktop App** | `window.electronAPI` (auth only) | No direct IPC in onboarding wizard itself |

## Implementation Notes

To replicate this onboarding system in OpenDroid:

1. **State Machine:** Implement a 4-step wizard using React `useState` or XState. Steps: `SIGNUP_WARNING` → `NAME` → `PAYMENT` → `FINAL`.

2. **Auth Gate:** Create a route guard that checks `organization.onboarded` and redirects unauthenticated/un-onboarded users to `/onboarding`.

3. **Video Tutorial:**
   - Create 5 MP4 assets (2 variants for Step 1: dark/light; 1 each for Steps 2–4).
   - Use `<video autoPlay loop muted playsInline>` for seamless inline playback.
   - Implement theme-aware selection: `videoMap[theme][stepNumber]`.
   - Only Step 1 needs theme variants; Steps 2–4 can be neutral.

4. **Theme System:**
   - Store preference in `localStorage` under `opendroid-theme-preference`.
   - Sync to DOM via `data-theme` attribute on `<html>`.
   - Default to `dark`.

5. **Backend Integration:**
   - `POST /api/organization/create`: accepts `{name, signupTouch}`.
   - `POST /api/organization/onboarding/complete`: accepts `{promotionId?, grantIntent}`.
   - `GET /api/auth/me`: must return `organization.onboarded` boolean.

6. **Payment Flow:**
   - Generate Stripe checkout URLs with `returnUrl` and `cancelUrl` pointing back to `/onboarding`.
   - Open Stripe in external browser (`window.open(url, "_blank")`).
   - Skip payment if promo has `skipPaymentMethod: true`.

7. **Completion Redirect:** After `onboarding/complete` API succeeds, invalidate auth cache and redirect to `/onboarding/finish?tier={paid|free}&origin={cli|web}`.

8. **Tutorial Modal:**
   - Non-dismissible modal (`onClose: noop`).
   - 7 slides: 5 video slides + 2 static (welcome + feedback).
   - On completion, persist `tutorialModalCompleted: true` flag.
   - Trigger condition in layout: `!!(shouldShowTutorial && !tutorialCompleted)`.

9. **Analytics:** Track each step transition with `gtag("event", "onboarding_{step}", metadata)`.

## Module Reference

| File | Offset Range | Symbol / Function | Notes |
|------|--------------|-------------------|-------|
| `renderer-main.js` | 9296–9301 | `bNs`, `vNs`, `jMn`, `BMn`, `UMn`, `yNs` | Video asset URL definitions + theme map |
| `renderer-main.js` | ~9966 | `$_` enum | Onboarding step keys (15 values, 4 used) |
| `renderer-main.js` | ~9967–9970 | `eQs()` | API config: `POST /api/organization/onboarding/complete` |
| `renderer-main.js` | ~9970–9973 | `tQs()` | API config: `POST /api/organization/create` |
| `renderer-main.js` | ~9973–9976 | `nQs()` | Returns base URL (`https://app.opendroid.dev` or `window.location.origin`) |
| `renderer-main.js` | ~9977–10050 | `rQs({isVerifiedMethod, search, promotionState})` | Main onboarding wizard state machine |
| `renderer-main.js` | ~10050+ | `sQs()` | Onboarding route wrapper: validates promo, determines auth method |
| `renderer-main.js` | ~10050+ | `aQs()` | Onboarding finish page: links to IDE extensions, CLI, desktop |
| `renderer-main.js` | ~10539 | `t0o(t)` | Tutorial modal: 7-slide carousel with videos |
| `renderer-main.js` | ~10520–10538 | `a0o` layout | Main app layout: triggers tutorial modal conditionally |
| `renderer-main.js` | ~10540–10560 | `$0o({children, data, error})` | Auth/onboarding gate: redirects based on `onboarded` flag |
| `renderer-main.js` | ~9966+ | `xB()` | Theme hook: reads/writes `opendroid-theme-preference` |
| `renderer-main.js` | ~9966+ | `SVs()`, `EVs()`, `syt(t)` | System theme detection, storage read, DOM sync |
| `renderer-main.js` | ~10050+ | `iQs(t)` | Promotion validation: `POST /api/promotions/{t}/validate` |
| `renderer-main.js` | ~9966+ | `BG(t, e)` | Google Analytics wrapper: `gtag("event", "onboarding_{t}", e)` |
