# gui-theme-css: GUI Architecture Notes

## Overview

The OpenDroid Electron GUI theme is defined entirely in a single 29KB CSS file (`theme.css`). It uses a **custom-CSS-first** architecture built around CSS custom properties (variables) scoped to `:root` and a `[data-theme=dark]` attribute selector for dark-mode overrides. The file bundles third-party library styles (xterm.js, allotment split-pane, cmdk command palette) alongside application-specific component selectors that rely heavily on `data-*` attributes. Two custom font families are loaded via `@font-face`: **Geist** (sans-serif, weights 400–700) and **Berkeley Mono** (monospace, weights 400–700). No Tailwind CSS directives or utility-class patterns are present in the stylesheet.

## File Map

| File | Size | Role |
|------|------|------|
| `theme.css` | 29 KB | Complete GUI theme: custom properties, dark/light mode, fonts, resets, component styles, and third-party library CSS |

## Architecture

The stylesheet is organized into the following conceptual layers (as they appear in the minified file):

1. **Design Token Layer**: `:root` custom properties for colors, spacing, radius, and typography.
2. **Theme Override Layer**: `[data-theme=dark]` selector that re-maps the semantic color tokens.
3. **Font Definition Layer**: `@font-face` rules for Geist and Berkeley Mono.
4. **Global Reset & Base Layer**: `*`, `html`, `body`, `code`, `pre`, `a`, `input` rules.
5. **Application Component Layer**: `data-*` attribute selectors for wiki rendering, sidebar, cmdk palette, mermaid diagrams, and terminal embeds.
6. **Third-Party Library Layer**: `.xterm`, `.allotment-module_*`, `.sash-module_*`, `cmdk-*` selectors imported from external dependencies.
7. **Animation Layer**: `@keyframes` for typewriter cursor, sidebar pan transitions, wiki heading highlight, and wiki version fold-out.

## Key Findings

- **Total custom properties**: ~85 variables across color, spacing, radius, and typography.
- **Dark mode mechanism**: Purely attribute-based (`[data-theme=dark]`): no `prefers-color-scheme` media query is present in the CSS.
- **Transition strategy**: The `body` element has a `transition` on `background-color`, `border-color`, `color`, and `opacity` (duration `.2s linear`), giving smooth theme switches when the `data-theme` attribute changes.
- **No Tailwind usage**: No `@tailwind`, `@apply`, or responsive/state-variant utility patterns exist in the CSS file.
- **Palette naming convention**: Colors use mineral/stone names (`tungsten`, `ruby`, `mica`, `topaz`, `jade`, `azurite`) rather than generic `gray-100`, `red-500`, etc.
- **Sidebar tokens**: Dedicated semantic variables (`sidebar-bg`, `sidebar-hover`, `sidebar-active`, `sidebar-text`, etc.) decouple sidebar chrome from generic surface colors.
- **Third-party CSS included inline**: xterm.js, allotment (split pane), and cmdk styles are bundled directly into the same file, suggesting the build tool inlines dependency CSS.

## Code Examples

### Light-mode `:root` token block (excerpt)
```css
:root {
  --primary-1: #f27b2f;
  --primary-2: #342f2d;
  --primary-10: #161413;
  --text-default: #000000;
  --text-inverted: #f2f0f0;
  --text-heading: #000000;
  --text-subheading: #736864;
  --text-muted: #b3a9a4;
  --text-label: #1d1b1a;
  --text-success: #5b8e63;
  --text-warning: #f0a330;
  --text-error: #e54048;
  --text-link: #157ac6;
  --text-highlight: #f27b2f;
  --surface-1: #f2f0f0;
  --surface-2: #e4e2e1;
  --surface-3: #d8d4d2;
  --surface-4: #c8c5c2;
  --surface-5: #bfb7b3;
  --surface-inverted: #161413;
  --border-1: #9b8e87;
  --border-2: #59514d;
  --border-3: #000000;
  --border-4: #a89895;
  --icon-accent: #bc4b00;
  --sidebar-bg: #f2f0f0;
  --sidebar-hover: #e4e2e1;
  --sidebar-active: #e4e2e1;
  --sidebar-active-text: #000000;
  --sidebar-text: #736864;
  --sidebar-text-hover: #1d1b1a;
  --sidebar-notification: #ee6018;
  --landing-logo: #4c4542;
  --spacing-none: 0px;
  --spacing-xs: 2px;
  --spacing-sm: 4px;
  --spacing-md: 8px;
  --spacing-lg: 12px;
  --spacing-xl: 16px;
  --spacing-2xl: 24px;
  --spacing-3xl: 32px;
  --spacing-4xl: 64px;
  --radius-none: 0px;
  --radius-xs: 2px;
  --radius-sm: 4px;
  --radius-md: 12px;
  --radius-lg: 24px;
  --radius-full: 9999px;
  --font-size-3xs: .5rem;
  --font-size-2xs: .625rem;
  --font-size-xs: .75rem;
  --font-size-sm: .8125rem;
  --font-size-md: .875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.125rem;
  --font-size-xl: 1.25rem;
  --font-size-2xl: 1.5rem;
  --font-size-3xl: 2.25rem;
}
```

### Dark-mode override block
```css
[data-theme=dark] {
  --primary-1: #d56a26;
  --primary-2: #e4e2e1;
  --primary-10: #ffa469;
  --text-default: #f2f0f0;
  --text-inverted: #000000;
  --text-heading: #f2f0f0;
  --text-subheading: #a89895;
  --text-muted: #b3a9a4;
  --text-label: #bfb7b3;
  --text-success: #5b8e63;
  --text-warning: #e3992a;
  --text-error: #d9363e;
  --text-link: #50acf2;
  --text-highlight: #d56a26;
  --surface-1: #161413;
  --surface-2: #1d1b1a;
  --surface-3: #282523;
  --surface-4: #342f2d;
  --surface-5: #403a37;
  --surface-inverted: #f2f0f0;
  --border-1: #342f2d;
  --border-2: #a89895;
  --border-3: #f2f0f0;
  --border-4: #4c4542;
  --icon-accent: #ee6018;
  --sidebar-bg: #1d1b1a;
  --sidebar-hover: #282523;
  --sidebar-active: #282523;
  --sidebar-active-text: #f2f0f0;
  --sidebar-text: #a89895;
  --sidebar-text-hover: #f2f0f0;
  --sidebar-notification: #f0a330;
  --landing-logo: #d56a26;
}
```

### Body transition rule (theme switch smoothing)
```css
body {
  min-height: 100dvh;
  background-color: var(--surface-1);
  color: var(--text-default);
  font-family: var(--font-sans);
  letter-spacing: -.13px;
  transition: background-color .2s linear, border-color .2s linear,
              color .2s linear, opacity .2s linear;
}
```

## Theme / Style Tokens

### CSS Custom Properties: Full Table

| Name | Light Value | Dark Value | Category |
|------|-------------|------------|----------|
| `--primary-1` | `#f27b2f` | `#d56a26` | Color / Semantic |
| `--primary-2` | `#342f2d` | `#e4e2e1` | Color / Semantic |
| `--primary-10` | `#161413` | `#ffa469` | Color / Semantic |
| `--text-default` | `#000000` | `#f2f0f0` | Color / Semantic |
| `--text-inverted` | `#f2f0f0` | `#000000` | Color / Semantic |
| `--text-heading` | `#000000` | `#f2f0f0` | Color / Semantic |
| `--text-subheading` | `#736864` | `#a89895` | Color / Semantic |
| `--text-muted` | `#b3a9a4` | `#b3a9a4` | Color / Semantic |
| `--text-label` | `#1d1b1a` | `#bfb7b3` | Color / Semantic |
| `--text-success` | `#5b8e63` | `#5b8e63` | Color / Semantic |
| `--text-warning` | `#f0a330` | `#e3992a` | Color / Semantic |
| `--text-error` | `#e54048` | `#d9363e` | Color / Semantic |
| `--text-link` | `#157ac6` | `#50acf2` | Color / Semantic |
| `--text-highlight` | `#f27b2f` | `#d56a26` | Color / Semantic |
| `--surface-1` | `#f2f0f0` | `#161413` | Color / Semantic |
| `--surface-2` | `#e4e2e1` | `#1d1b1a` | Color / Semantic |
| `--surface-3` | `#d8d4d2` | `#282523` | Color / Semantic |
| `--surface-4` | `#c8c5c2` | `#342f2d` | Color / Semantic |
| `--surface-5` | `#bfb7b3` | `#403a37` | Color / Semantic |
| `--surface-inverted` | `#161413` | `#f2f0f0` | Color / Semantic |
| `--border-1` | `#9b8e87` | `#342f2d` | Color / Semantic |
| `--border-2` | `#59514d` | `#a89895` | Color / Semantic |
| `--border-3` | `#000000` | `#f2f0f0` | Color / Semantic |
| `--border-4` | `#a89895` | `#4c4542` | Color / Semantic |
| `--icon-accent` | `#bc4b00` | `#ee6018` | Color / Semantic |
| `--sidebar-bg` | `#f2f0f0` | `#1d1b1a` | Color / Semantic |
| `--sidebar-hover` | `#e4e2e1` | `#282523` | Color / Semantic |
| `--sidebar-active` | `#e4e2e1` | `#282523` | Color / Semantic |
| `--sidebar-active-text` | `#000000` | `#f2f0f0` | Color / Semantic |
| `--sidebar-text` | `#736864` | `#a89895` | Color / Semantic |
| `--sidebar-text-hover` | `#1d1b1a` | `#f2f0f0` | Color / Semantic |
| `--sidebar-notification` | `#ee6018` | `#f0a330` | Color / Semantic |
| `--landing-logo` | `#4c4542` | `#d56a26` | Color / Semantic |
| `--tungsten-1` | `#f2f0f0` |: | Color / Palette |
| `--tungsten-2` | `#e4e2e1` |: | Color / Palette |
| `--tungsten-3` | `#d8d4d2` |: | Color / Palette |
| `--tungsten-4` | `#c8c5c2` |: | Color / Palette |
| `--tungsten-5` | `#bfb7b3` |: | Color / Palette |
| `--tungsten-6` | `#b3a9a4` |: | Color / Palette |
| `--tungsten-7` | `#a89895` |: | Color / Palette |
| `--tungsten-8` | `#9b8e87` |: | Color / Palette |
| `--tungsten-9` | `#948781` |: | Color / Palette |
| `--tungsten-10` | `#80756f` |: | Color / Palette |
| `--tungsten-11` | `#736864` |: | Color / Palette |
| `--tungsten-12` | `#665c58` |: | Color / Palette |
| `--tungsten-13` | `#59514d` |: | Color / Palette |
| `--tungsten-14` | `#4c4542` |: | Color / Palette |
| `--tungsten-15` | `#403a37` |: | Color / Palette |
| `--tungsten-16` | `#342f2d` |: | Color / Palette |
| `--tungsten-17` | `#282523` |: | Color / Palette |
| `--tungsten-18` | `#1d1b1a` |: | Color / Palette |
| `--tungsten-19` | `#161413` |: | Color / Palette |
| `--tungsten-20` | `#000000` |: | Color / Palette |
| `--ruby-1` | `#c43037` |: | Color / Palette |
| `--ruby-2` | `#d9363e` |: | Color / Palette |
| `--ruby-3` | `#e54048` |: | Color / Palette |
| `--ruby-4` | `#ff9494` |: | Color / Palette |
| `--mica-1` | `#bc4b00` |: | Color / Palette |
| `--mica-2` | `#ed4823` |: | Color / Palette |
| `--mica-3` | `#ee6018` |: | Color / Palette |
| `--mica-4` | `#d56a26` |: | Color / Palette |
| `--mica-5` | `#f27b2f` |: | Color / Palette |
| `--mica-6` | `#ffa469` |: | Color / Palette |
| `--mica-7` | `#ebb28c` |: | Color / Palette |
| `--topaz-1` | `#b1761e` |: | Color / Palette |
| `--topaz-2` | `#e3992a` |: | Color / Palette |
| `--topaz-3` | `#f0a330` |: | Color / Palette |
| `--jade-1` | `#4a7450` |: | Color / Palette |
| `--jade-2` | `#5b8e63` |: | Color / Palette |
| `--jade-3` | `#6fab78` |: | Color / Palette |
| `--azurite-1` | `#157ac6` |: | Color / Palette |
| `--azurite-2` | `#378bc8` |: | Color / Palette |
| `--azurite-3` | `#50acf2` |: | Color / Palette |
| `--spacing-none` | `0px` |: | Spacing |
| `--spacing-xs` | `2px` |: | Spacing |
| `--spacing-sm` | `4px` |: | Spacing |
| `--spacing-md` | `8px` |: | Spacing |
| `--spacing-lg` | `12px` |: | Spacing |
| `--spacing-xl` | `16px` |: | Spacing |
| `--spacing-2xl` | `24px` |: | Spacing |
| `--spacing-3xl` | `32px` |: | Spacing |
| `--spacing-4xl` | `64px` |: | Spacing |
| `--radius-none` | `0px` |: | Border Radius |
| `--radius-xs` | `2px` |: | Border Radius |
| `--radius-sm` | `4px` |: | Border Radius |
| `--radius-md` | `12px` |: | Border Radius |
| `--radius-lg` | `24px` |: | Border Radius |
| `--radius-full` | `9999px` |: | Border Radius |
| `--font-size-3xs` | `.5rem` |: | Typography |
| `--font-size-2xs` | `.625rem` |: | Typography |
| `--font-size-xs` | `.75rem` |: | Typography |
| `--font-size-sm` | `.8125rem` |: | Typography |
| `--font-size-md` | `.875rem` |: | Typography |
| `--font-size-base` | `1rem` |: | Typography |
| `--font-size-lg` | `1.125rem` |: | Typography |
| `--font-size-xl` | `1.25rem` |: | Typography |
| `--font-size-2xl` | `1.5rem` |: | Typography |
| `--font-size-3xl` | `2.25rem` |: | Typography |
| `--font-sans` | `"Geist", -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif` |: | Typography |
| `--font-mono` | `"Berkeley Mono", "SF Mono", Monaco, Inconsolata, "Roboto Mono", Consolas, "Liberation Mono", Courier, monospace` |: | Typography |

## Dark / Light Mode Mechanism

The theme uses an **attribute-based toggle** rather than a media-query or class-based approach:

- **Selector**: `[data-theme=dark]` overrides the `:root` custom properties.
- **Scope**: The attribute is expected to be set on a top-level ancestor (most likely `<html>` or `<body>`), because `[data-theme=dark]` is a plain attribute selector without a specific element tag.
- **Transition**: The `body` element declares:
  ```css
  transition: background-color .2s linear, border-color .2s linear,
              color .2s linear, opacity .2s linear;
  ```
  This ensures that when `data-theme` is flipped by JavaScript, all color-dependent properties animate smoothly.
- **No `prefers-color-scheme`**: There is **no** `@media (prefers-color-scheme: dark)` block in the CSS. System-level dark-mode preference must be handled by the JavaScript layer (React) which sets the `data-theme` attribute accordingly.

## Font-Family Declarations

### Sans-serif: Geist
Loaded at four weights from local `.woff2` files:

| Weight | File | Style |
|--------|------|-------|
| 400 (Regular) | `Geist-Regular.woff2` | normal |
| 500 (Medium) | `Geist-Medium.woff2` | normal |
| 600 (SemiBold) | `Geist-SemiBold.woff2` | normal |
| 700 (Bold) | `Geist-Bold.woff2` | normal |

**Fallback stack**: `-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif`

### Monospace: Berkeley Mono
Loaded at four weights from local `.woff2` files:

| Weight | File | Style |
|--------|------|-------|
| 400 (Regular) | `Mono-Regular.woff2` | normal |
| 500 (Medium) | `Mono-Medium.woff2` | normal |
| 600 (SemiBold) | `Mono-SemiBold.woff2` | normal |
| 700 (Bold) | `Mono-Bold.woff2` | normal |

**Fallback stack**: `"SF Mono", Monaco, Inconsolata, "Roboto Mono", Consolas, "Liberation Mono", Courier, monospace`

## Tailwind Patterns Audit

| Pattern | Evidence in CSS | Assessment |
|---------|----------------|------------|
| `@tailwind` directives | **None found** | No Tailwind compilation directives present |
| `@apply` rules | **None found** | No utility-class extraction into custom selectors |
| `dark:` variant utilities | **None found** | Dark mode is handled via `[data-theme=dark]` overriding `:root` variables, not via Tailwind's `dark:` prefix |
| Responsive prefixes (`sm:`, `md:`, `lg:`, `xl:`) | **None found** | No responsive utility classes in CSS |
| State variants (`hover:`, `focus:`, `active:`) | **None found** | Hover/focus states are written as standard pseudo-class selectors (e.g., `[cmdk-item]:hover`) |

**Conclusion**: The CSS file is **custom-CSS-first**. There is zero evidence that Tailwind CSS is being used as the primary styling framework. The design system is hand-authored with CSS custom properties and component-specific selectors. Any Tailwind-like class names that may appear in JSX (e.g., `className="dark:text-white"`) would be a separate concern outside this file; the compiled stylesheet itself contains only custom CSS.

## Theme Architecture Assessment

### Custom-CSS-First (Confirmed)

1. **Token-driven**: All colors, spacing, radius, and typography are centralized in `:root` variables, making the theme highly maintainable and enabling the dark-mode switch by simply re-mapping those variables.
2. **Semantic + Palette hybrid**: The system uses semantic names (`--text-default`, `--surface-1`) for application surfaces and text, while retaining mineral-named palettes (`--tungsten-5`, `--ruby-3`) for fine-grained or legacy usage.
3. **Component isolation via data attributes**: Custom components are scoped with `data-*` attributes (e.g., `[data-wiki-markdown]`, `[data-sidebar-anim]`, `[data-mermaid-container]`), which keeps specificity predictable and avoids class-name collision.
4. **Third-party CSS co-location**: Library styles (xterm.js, allotment, cmdk) are inlined into the same compiled file. This is likely a Vite build artifact and means the theme is tightly coupled to the exact versions of those libraries at build time.
5. **No utility-class layer**: Because there are no utility classes, every component style must be explicitly authored. This implies a larger stylesheet but deterministic styling.

## IPC Channels

*N/A for this feature: the CSS file contains no JavaScript or IPC references.*

## Integration Points

- **Renderer bundle (`renderer-main.js`)**: The JS layer is responsible for toggling the `data-theme` attribute and referencing these CSS variables via inline styles or computed values. The dark/light logic lives in React, not CSS.
- **TUI Theme (01-terminal-ui scope)**: The color palette names (`tungsten`, `ruby`, `mica`, `topaz`, `jade`, `azurite`) may mirror or conflict with TUI theme tokens. Cross-mission comparison recommended.
- **Font binaries**: The `@font-face` declarations reference `.woff2` files present in the same `assets/` directory. Font loading is handled by the browser renderer.
- **xterm.js / allotment / cmdk**: These library styles are bundled here but their behavioral logic is in the renderer JS bundles (separate features).

## Implementation Notes

1. **Theme tokens**: Copy the complete `:root` block and `[data-theme=dark]` override block into OpenDroid's global CSS. The token names are well-structured and can be mapped 1:1 to any CSS-in-JS or Tailwind theme extension.
2. **Dark mode**: Implement the same `[data-theme="dark"]` attribute toggle in OpenDroid's `<html>` or `<body>` tag. Use the existing `transition` on `body` for smooth switching.
3. **Fonts**: Ship the same Geist and Berkeley Mono `.woff2` files (or substitute with open-source alternatives if licensing requires). Register identical `@font-face` rules.
4. **Spacing & radius scale**: The `--spacing-*` and `--radius-*` scales are simple and can be reused directly or mapped to a design-system config (e.g., Tailwind `theme.extend`).
5. **Component styles**: The `data-*` selectors can be converted to CSS Modules, styled-components, or Tailwind arbitrary variants depending on OpenDroid's chosen architecture.
6. **Third-party CSS**: For xterm.js, allotment, and cmdk, prefer importing their official CSS via npm rather than copying the inline styles, then override with the custom property tokens where needed.

## Module Reference

| File | Offset Range | Symbol / Function | Notes |
|------|--------------|-------------------|-------|
| `theme.css` | 0:– | `:root` light tokens | Full file read (29KB minified) |
| `theme.css` | 0:– | `[data-theme=dark]` overrides | Dark mode token re-mapping |
| `theme.css` | 0:– | `@font-face` (Geist) | 400/500/600/700 weights |
| `theme.css` | 0:– | `@font-face` (Berkeley Mono) | 400/500/600/700 weights |
| `theme.css` | 0:– | `body` base rule | `transition` for theme smoothing |
| `theme.css` | 0:– | `.xterm` rules | xterm.js terminal styles |
| `theme.css` | 0:– | `allotment-module_*` | Split-pane library styles |
| `theme.css` | 0:– | `sash-module_*` | Sash drag-handle styles |
| `theme.css` | 0:– | `[cmdk-*]` / `.cmdk-*` | cmdk command palette styles |
| `theme.css` | 0:– | `[data-wiki-*]` | Custom wiki/markdown component styles |
| `theme.css` | 0:– | `[data-sidebar-*]` | Custom sidebar animation & layout styles |
| `theme.css` | 0:– | `@keyframes` | Typewriter blink, sidebar pan, wiki highlight, version fold |
