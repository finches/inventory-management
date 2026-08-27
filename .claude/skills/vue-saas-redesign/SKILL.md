---
name: vue-saas-redesign
description: Redesign a Vue 3 app's top navigation into a modern SaaS-style left sidebar layout with consistent design tokens (spacing/color/radius/shadow) and a polished look. Use when asked to redesign the UI, convert to a sidebar layout, modernize the navigation, or give the app a SaaS look.
---

# Vue 3 SaaS Sidebar Redesign

Converts a Vue 3 app's horizontal top navigation into a modern SaaS-style left sidebar layout, introduces a consistent design-token scale (spacing, color, radius, shadow), and polishes the overall visual language. Written to be generic — it works on any Vue 3 codebase — with this repo's `inventory-management` app used throughout as a concrete worked example.

## When to Use This Skill

- "Redesign the UI" / "modernize the navigation" / "convert to a sidebar layout" / "give this a SaaS look"
- Any request to replace a top nav bar with a left-side vertical nav
- Any request to make spacing/styling more consistent across an existing Vue app

There is no separate approval-gated planning step baked into this skill — once discovery (Step 1) is done, proceed straight to implementation (Step 5). If the invoking context is itself a plan-mode session, follow that session's plan workflow instead; otherwise, execute directly.

## Step 1 — Discover the Current Layout

Before changing anything, establish facts about the target app:

1. **Find the root shell component.** Look for `App.vue` (or `Layout.vue`, `MainLayout.vue`, `DefaultLayout.vue`) — usually imported directly in `main.js`/`main.ts`. This is almost always where the top nav lives.
2. **Find the router config.** Locate `createRouter(...)` (commonly `src/router/index.js` or `src/main.js`) to enumerate routes for nav items. Check whether routes declare `name`/`meta.icon`/`meta.label` — if not, note where labels currently come from (i18n composable, hardcoded template strings, etc.).
3. **Find the nav markup and active-link logic.** Search for `<nav`, `router-link`, `.active`, `$route.path ===`. Note whether the app already uses Vue Router's built-in `router-link-active` / `active-class`, or manual `:class="{ active: ... }"` bindings that need to be ported per-item.
4. **Find hardcoded sticky/offset couplings.** Grep for `position: sticky`, `position: fixed`, and numeric `top:` values. Any secondary bar whose `top` matches the nav's pixel height is a load-bearing coupling that assumes a horizontal nav sits above it — removing the top nav will break it unless it's fixed.
5. **Find or confirm absence of design tokens.** Grep for `--` custom properties and for a global stylesheet (`main.css`, `tokens.css`, `variables.css`). If none exist, shared styles may instead live inside the root shell's unscoped `<style>` block and leak globally via CSS specificity rather than imports — confirm before assuming a design-system file exists.
6. **Inventory the current visual language.** Pull the actual hex colors, spacing values, border-radius, and shadow values already in use so the new token scale is derived from the app's existing palette, not an arbitrary one.
7. **Check for an icon system.** Check `package.json` for an icon library (Heroicons, Lucide, Font Awesome, etc.). If none, icons are likely hand-written inline SVGs — default to continuing that convention rather than adding a new dependency unless one is already present.
8. **Identify the delegation convention.** Read the project's `CLAUDE.md` for a mandatory subagent-delegation rule for `.vue` file changes (e.g. "any `.vue` file creation/modification must go through `<agent-name>`"), and note that subagent's declared file scope.

> **What you'll find in this repo (inventory-management):**
>
> - Root shell: `client/src/App.vue` — `.app { flex-direction: column }`, a sticky `<header class="top-nav">` at `height: 70px` containing `.nav-tabs` (manual `:class="{ active: $route.path === '/x' }"` router-links), `LanguageSwitcher`, and `ProfileMenu` pushed right via `margin-left: auto`.
> - Router: `client/src/main.js` — 6 routes registered inline, no `name`/`meta` fields, no lazy `import()`.
> - Coupling hazard: `client/src/components/FilterBar.vue` has `.filters-bar { position: sticky; top: 70px; z-index: 90; }` — hardcoded to the nav's exact height.
> - Zero CSS custom properties anywhere; no global stylesheet — shared classes (`.page-header`, `.card`, `.stat-card`, `.badge`, table styles) live in `App.vue`'s unscoped `<style>` and leak to all views.
> - No icon library in `package.json` (only `vue`, `vue-router`, `axios`) — icons are inline hand-written SVGs.
> - Delegation rule in `CLAUDE.md`: any `.vue` file creation/modification must go through the `vue-expert` subagent, scoped to `client/src/views/*.vue`, `client/src/components/*.vue`, `client/src/composables/*.js`, `client/src/api.js`, `client/src/App.vue`, `client/src/main.js`.

## Step 2 — Establish the Design Token Scale

Introduce a small, consistent set of CSS custom properties so spacing/color/radius/shadow stop being repeated magic values. Place them in a new global stylesheet if the project has none (e.g. `src/styles/tokens.css`, imported once in `main.js`), or at the top of the existing global `<style>` block if that's the project's established convention — follow whatever Step 1 found.

```css
:root {
  /* Spacing (4px base unit) */
  --space-1: 0.25rem; /* 4px */
  --space-2: 0.5rem; /* 8px */
  --space-3: 0.75rem; /* 12px */
  --space-4: 1rem; /* 16px */
  --space-5: 1.25rem; /* 20px */
  --space-6: 1.5rem; /* 24px */
  --space-8: 2rem; /* 32px */

  /* Radius */
  --radius-sm: 6px;
  --radius-md: 10px;
  --radius-lg: 14px;

  /* Shadow */
  --shadow-sm: 0 1px 3px 0 rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 12px rgba(0, 0, 0, 0.06);
  --shadow-lg: 0 10px 25px rgba(0, 0, 0, 0.08);

  /* Color — derive from the app's EXISTING palette, don't invent a new one */
  --color-bg: #f8fafc;
  --color-surface: #ffffff;
  --color-border: #e2e8f0;
  --color-text-primary: #0f172a;
  --color-text-secondary: #64748b;
  --color-accent: #2563eb;
  --color-accent-soft: #eff6ff;

  /* Sidebar-specific */
  --sidebar-width: 260px;
  --sidebar-width-collapsed: 72px;
}
```

Retrofit existing shared classes (`.card`, `.stat-card`, `.badge`, tables, etc.) to use these tokens instead of hardcoded hex/rem values as part of the same pass — this is what "consistent spacing" means in practice, not just for the new sidebar but across the whole app.

> **This repo:** the color tokens above are pulled directly from `App.vue`'s existing hex values (`#0f172a`, `#64748b`, `#e2e8f0`, `#2563eb`, `#f8fafc`) so the redesign keeps the same brand feel.

## Step 3 — Target Sidebar Layout Spec

**Shell restructure:** change the root container from a column flex (nav on top, content below) to a horizontal grid/flex: `display: grid; grid-template-columns: var(--sidebar-width) 1fr;`, with the sidebar as a fixed-width column and an independently scrolling content column.

**Sidebar anatomy (top to bottom):**

1. **Brand/logo zone** — top of sidebar, fixed height, matches the old nav's logo treatment.
2. **Primary nav list** — vertical list, one row per route, icon + label.
3. **(Optional) secondary/utility nav** — settings/help, visually separated by a divider or pushed down via `margin-top: auto`.
4. **Footer/profile zone** — pinned to the bottom via `margin-top: auto`, houses whatever lived on the right side of the old top nav (profile menu, language switcher).

**Active state:** full-row background highlight (not an underline — underlines read as "tab," not "sidebar item"): `var(--color-accent-soft)` background, `var(--color-accent)` text/icon, optionally a left accent bar for a stronger vertical-nav affordance.

**Hover state:** a subtle neutral background distinct from the active state.

**Collapsible behavior:** a toggle that shrinks the sidebar to `--sidebar-width-collapsed` (icon-only, labels hidden), animated via `transition: width 0.2s ease`.

**Responsive behavior:** define at least one breakpoint (e.g. `768px`) below which the sidebar becomes an off-canvas drawer (`transform: translateX(-100%)` by default, toggled via a hamburger button in a slim mobile-only top bar). If the app has no existing breakpoints to match, pick reasonable common values (`640px`, `1024px`) rather than skipping responsive behavior entirely.

**Relocating right-aligned nav widgets:** move anything that lived on the far right of the old top nav (language switcher, notifications, profile menu) into the sidebar footer if there are only 1-2 such widgets; keep a slim top utility bar for 3+ widgets, with all primary navigation still fully moved into the sidebar.

```css
.app-shell {
  display: grid;
  grid-template-columns: var(--sidebar-width) 1fr;
  min-height: 100vh;
}

.sidebar {
  background: var(--color-surface);
  border-right: 1px solid var(--color-border);
  display: flex;
  flex-direction: column;
  padding: var(--space-4);
  gap: var(--space-2);
}

.sidebar-brand {
  padding: var(--space-2) var(--space-3);
  margin-bottom: var(--space-4);
}

.sidebar-nav {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
}

.sidebar-nav-item {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-3);
  border-radius: var(--radius-sm);
  color: var(--color-text-secondary);
  text-decoration: none;
  font-weight: 500;
  transition:
    background-color 0.15s ease,
    color 0.15s ease;
}

.sidebar-nav-item:hover {
  background: var(--color-bg);
  color: var(--color-text-primary);
}

.sidebar-nav-item.router-link-active {
  background: var(--color-accent-soft);
  color: var(--color-accent);
  font-weight: 600;
}

.sidebar-footer {
  margin-top: auto;
  border-top: 1px solid var(--color-border);
  padding-top: var(--space-3);
}

.content-area {
  overflow-y: auto;
  padding: var(--space-6) var(--space-8);
}
```

Prefer Vue Router's built-in `router-link-active` / `exact-active-class` (or the `active-class` prop) over manual `$route.path === '/x'` comparisons wherever the target app used the manual pattern — this is both a modernization and a simplification.

## Step 4 — Fix Sticky/Offset Couplings

Any element found in Step 1.4 with a hardcoded `top: <nav-height>px` needs to be re-evaluated now that no horizontal nav sits above the content column. It typically becomes `top: 0` once it's the topmost sticky element inside the scrollable content area — but confirm based on where it now lives in the DOM tree.

> **This repo:** `FilterBar.vue`'s `.filters-bar { position: sticky; top: 70px; }` must change to `top: 0` once `FilterBar` renders inside the new `.content-area`. Verify by scrolling a data-heavy view (e.g. `/orders`) and confirming the filter bar sticks flush to the top with no gap or overlap.

## Step 5 — Delegate Implementation

- Re-check the project's `CLAUDE.md` (found in Step 1.8) for a mandatory subagent-delegation rule before editing any `.vue` file.
- If a rule exists: delegate all in-scope file changes to that subagent via the Task tool, in a few well-scoped calls rather than one per file — e.g. (1) shell/router restructure into the sidebar, (2) shared-style token retrofit plus the Step 4 coupling fixes, (3) responsive/collapse behavior. Give the subagent the token scale, sidebar spec, and coupling fixes explicitly in the prompt — it likely has no built-in SaaS-sidebar guidance of its own.
- If no such rule exists in the target project: edit the files directly, following whatever conventions Step 1 discovered (Composition API vs. Options API, scoped vs. global CSS, existing naming patterns).
- Do not insert a human approval gate between discovery and delegation — Steps 1-4 above are the plan; move straight to implementation once they're done.

> **This repo:** delegate to `vue-expert` (scope: `client/src/views/*.vue`, `client/src/components/*.vue`, `client/src/composables/*.js`, `client/src/api.js`, `client/src/App.vue`, `client/src/main.js`) with the 3-task breakdown above.

## Step 6 — Verify

1. Start the dev server per the project's documented command (check `CLAUDE.md`/`package.json` scripts, or a project `start`/`run` skill if one exists).
2. Use Playwright MCP tools to:
   - Screenshot the desktop view showing the new sidebar.
   - Resize to the mobile breakpoint from Step 3 and screenshot the collapsed/off-canvas state.
   - Click through several nav items and confirm the active-state highlight moves correctly.
   - Scroll views that had a sticky secondary bar (Step 4) and confirm no gap/overlap.
   - Check the browser console for errors/warnings — Vue prop/key warnings often surface template regressions from the restructure.
3. Confirm no visual regression in content that wasn't part of the redesign — spacing should look _more_ consistent after the token retrofit, not broken.

## Final Checklist

- [ ] Root shell restructured from top-nav layout to sidebar + content grid/flex
- [ ] Design tokens (spacing, color, radius, shadow) defined and applied to both the new sidebar and existing shared components
- [ ] All nav routes represented in the sidebar with icon + label; active state uses Router's built-in active classes where possible
- [ ] Profile/language/utility widgets relocated into the sidebar footer or a slim utility bar
- [ ] All hardcoded sticky/offset couplings found in Step 1 fixed
- [ ] Responsive breakpoint(s) added; sidebar collapses or goes off-canvas on narrow viewports
- [ ] All `.vue` edits routed through the project's mandated subagent, if one is declared in `CLAUDE.md`
- [ ] Playwright verification done: desktop + mobile screenshots, active-link check, console-error check, no layout shift on previously-sticky elements
- [ ] No new npm dependency added unless one was already present or explicitly justified
