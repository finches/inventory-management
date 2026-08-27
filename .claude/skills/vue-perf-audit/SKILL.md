---
name: vue-perf-audit
description: Analyze a Vue 3 app's component structure and produce a prioritized report of performance and code-reuse optimizations. Use when asked to audit Vue components, find performance issues, suggest refactors, identify duplicated logic/markup across components, or improve reactivity patterns. Written to be generic — works on any Vue 3 codebase, with this repo's inventory-management app used as a concrete worked example.
---

# Vue 3 Component Performance & Reuse Audit

Produces a prioritized, evidence-based report of performance and code-reuse issues across a Vue 3 codebase's components, views, and composables. This is an **analysis skill by default** — it does not modify code unless the user explicitly asks for fixes to be applied.

## When to Use This Skill

- "Audit the Vue components" / "find performance issues" / "what can we optimize"
- "Where's duplicated logic/markup across components" / "what should be a composable"
- Any request to review component structure before a refactor, without a specific bug in mind

## Step 1 — Inventory the Codebase

1. Find all `.vue` files (`find . -name "*.vue"`) and split them into views (`src/views/`) vs. reusable components (`src/components/`) vs. anything else.
2. Get a line-count ranking (`wc -l`) — the largest files are the first place to look for components doing too much and candidates for splitting.
3. Check for a `composables/` directory. Its absence, combined with repeated data-loading/filtering logic across views, is itself a finding (see Step 3).
4. Check the project's `CLAUDE.md` for a mandatory subagent-delegation rule for `.vue` file changes — needed only if this skill proceeds to Step 5 (applying fixes), not for the audit itself.

> **This repo:** 18 `.vue` files, ~8,200 lines total. `views/Dashboard.vue` (1,273 lines) and `views/Spending.vue` (852 lines) are the largest by far — both are prime candidates for extracting sub-components and composables. `client/src/composables/` exists (`useFilters`, `useAuth`-style composables) but data-fetching/loading-state boilerplate is still repeated per-view rather than centralized (see Step 3). No performance-testing tooling in `package.json` — audit is static/manual.

## Step 2 — Performance Checklist

Run each check across every `.vue` file found in Step 1. For each hit, record file, line, and a one-line description — don't just note the pattern name.

| Check                                                                                                                         | How to find it                                                                                                                                       | Why it matters                                                                                                                                                      |
| ----------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Index used as `:key` in `v-for`**                                                                                           | `grep -rn 'v-for=' ` then inspect the paired `:key`                                                                                                  | Causes Vue to misattribute DOM state (input focus, transitions, component instance) when the list reorders/filters/sorts — a correctness bug as much as a perf one. |
| **Heavy computation inside a method or inline template expression instead of `computed`**                                     | Look for `.filter()/.map()/.reduce()/.sort()` chains called directly in `<template>` or inside a plain function invoked from the template            | Re-runs on every render instead of only when dependencies change.                                                                                                   |
| **`watch`/`watchEffect` used where a `computed` would do**                                                                    | `grep -rln 'watch('` then check whether the watcher just sets another ref from a pure function of its source(s)                                      | Computed is cached and declarative; a watcher doing the same thing adds an extra render tick and is easier to get wrong (missed dependencies, stale closures).      |
| **Unbounced watchers on search/filter text input**                                                                            | Search-bound refs watched directly, feeding either a re-filter of a large array or an API call                                                       | Refilters/re-requests on every keystroke; needs `watchDebounced` (or manual debounce) per the project's own `CLAUDE.md` guidance if one exists.                     |
| **`v-if` on elements toggled very frequently (e.g. modals, tabs)**                                                            | Cross-reference toggle frequency against `v-if`/`v-show` usage                                                                                       | `v-if` destroys/recreates the subtree each toggle; `v-show` (CSS display toggle) is cheaper for frequent, cheap-to-render content.                                  |
| **Large/rarely-used components (e.g. modals) not lazy-loaded**                                                                | Check whether modal/detail components are statically imported in a view vs. `defineAsyncComponent(() => import(...))`                                | Every static import adds to the initial bundle even if the user never opens that modal.                                                                             |
| **New object/array literals created inline in template bindings** (`:style="{ ... }"`, `:class="[...]"`, inline prop objects) | Grep template sections for `:style="{`, `:class="[`                                                                                                  | New reference every render defeats child `shallowRef`/`memo`-style optimizations and can trigger unnecessary child re-evaluation.                                   |
| **Full-object deep reactivity on large, rarely-mutated collections**                                                          | Look for `ref([...bigArray])`/`reactive({ bigArray })` holding large API response arrays that are only ever replaced wholesale, never deeply mutated | `shallowRef` avoids Vue instrumenting every nested property of a large dataset it never needs to track deeply.                                                      |
| **Duplicate/overlapping API calls on mount** across sibling views for the same data                                           | Grep `api.js` call sites per view; look for the same endpoint fetched independently by multiple mounted views in the same session                    | Candidate for a shared composable with request caching/deduping instead of N independent fetches.                                                                   |

> **This repo — confirmed hits:**
>
> - `views/Reports.vue:28,51,82` — three separate `v-for` loops keyed on array `index` (`quarterlyData`, `monthlyData` twice) instead of a stable field like `quarter`/`month`. Per this repo's own `CLAUDE.md` "Common Issues" section, this is a known anti-pattern here.
> - All 7 modal components (`BacklogDetailModal`, `CostDetailModal`, `ProductDetailModal`, `PurchaseOrderModal`, `TasksModal`, `ProfileDetailsModal`, `InventoryDetailModal`) are statically imported by their parent views/`App.vue` even though only one is ever open at a time — all are lazy-loading candidates via `defineAsyncComponent`.
> - `views/Dashboard.vue` at 1,273 lines mixes KPI computation, chart data shaping, and table rendering in one file — check for `computed` chains that could be pushed into smaller child components to scope re-render impact.

## Step 3 — Code-Reuse Checklist

| Check                                                                                                                                               | How to find it                                                                                             | Why it matters                                                                                                                                                                                         |
| --------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Repeated data-loading boilerplate** (`loading`/`error`/`data` refs + try/catch/finally) copy-pasted per view                                      | Grep `const loading = ref` across `views/*.vue`                                                            | Extract into a `useAsyncData(fetcher)`-style composable; this repo's own `CLAUDE.md` documents this exact try/catch/finally pattern as the standard, meaning it's likely duplicated verbatim per view. |
| **Duplicated markup blocks** (stat cards, page headers, empty states, badges) copy-pasted across views instead of extracted into a shared component | Diff the template structure of similarly-styled blocks (e.g. `.stat-card`, `.page-header`) across 2+ files | Turns N places to update into 1; reduces visual drift over time.                                                                                                                                       |
| **Duplicated modal shell structure** (Teleport + Transition + backdrop + close button) reimplemented per modal                                      | Compare the top of each `*Modal.vue` file's template                                                       | Extract a `BaseModal.vue` wrapper that owns Teleport/Transition/backdrop/ESC-to-close, with each specific modal only supplying its body content via a slot.                                            |
| **Duplicated filter/derivation logic** (e.g. the same "filter items by category+warehouse" logic reimplemented per view instead of shared)          | Grep for repeated `.filter(item => item.category === ... && item.warehouse === ...)`-shaped chains         | Extract into a composable or shared utility function taking the raw list + filter state.                                                                                                               |
| **Prop drilling past 2 levels** for the same value (e.g. passing filters or user info through an intermediate component that doesn't use it)        | Trace props declared but only re-emitted/re-passed, not consumed, by the component that declares them      | Candidate for `provide`/`inject` or a shared composable instead of drilling.                                                                                                                           |
| **Duplicated CSS values instead of design tokens**                                                                                                  | Grep scoped `<style>` blocks for repeated raw hex colors / px values across files                          | If the project already has a design-token scale (e.g. from a prior redesign pass), flag any component still using raw hardcoded values instead of the tokens.                                          |

> **This repo — confirmed hits:**
>
> - 7 modal components each independently implement their own show/hide + backdrop pattern — strong `BaseModal.vue` extraction candidate (verify by reading 2-3 of them, e.g. `BacklogDetailModal.vue` and `CostDetailModal.vue`, and diffing their top-level template structure).
> - Every view under `src/views/` follows the `loading`/`error`/`data` + `try/catch/finally` pattern documented in `client/CLAUDE.md`'s "Data Loading Pattern" section verbatim — this is a strong signal it's copy-pasted rather than shared, since the pattern is written down as a convention but no composable enforces it.
> - CSS design tokens (`--color-*`, `--space-*`, etc.) were introduced by the `vue-saas-redesign` skill's token retrofit — check newer vs. older components for consistency now that both patterns may coexist.

## Step 4 — Produce the Report

Output a Markdown report (in the response, not necessarily a new file — only write one if the user asked for a persisted document) structured as:

```markdown
# Vue Component Audit — <app name>

## Summary

<N> performance findings, <M> reuse findings across <X> files.

## Performance

### High impact

- **<file>:<line>** — <issue>. <concrete fix>. Impact: <why this matters at this app's scale>.

### Medium / Low impact

- ...

## Code Reuse

### High impact

- **<files>** — <duplicated pattern>. <concrete extraction proposal — new composable/component name and shape>.

### Medium / Low impact

- ...

## Suggested Order of Work

1. <highest-leverage, lowest-risk item first>
2. ...
```

Rank by **impact × ease**, not just severity — e.g. fixing 3 `v-for` index keys is higher priority than a speculative `shallowRef` swap on data that's rarely re-rendered, even though both are "performance."

Do not invent findings to pad the report — if a checklist item in Step 2/3 has no real hits, omit it rather than listing a token low-severity nit.

## Step 5 — (Only If Asked) Apply Fixes

This skill stops at the report by default. Only proceed to implementation if the user explicitly asks to apply some or all suggestions.

- Re-check the project's `CLAUDE.md` for a mandatory subagent-delegation rule before editing any `.vue` file (this repo requires routing all `.vue` creation/modification through `vue-expert`).
- Batch related fixes into a few well-scoped delegation calls (e.g. one for all `v-for` key fixes, one for `BaseModal` extraction + migrating existing modals, one for the shared data-loading composable) rather than one call per file.
- After fixes are applied, verify: start the dev server, click through affected views, and check the browser console for new warnings/errors — a lazy-loading or composable-extraction change is exactly the kind of refactor that can silently break a prop or event wire-up.

## Final Checklist

- [ ] All `.vue` files inventoried with line counts; largest files flagged for structural review
- [ ] Every Step 2 performance check run against the full file list, with concrete file:line evidence for each hit (no speculative findings)
- [ ] Every Step 3 reuse check run, with at least 2 concrete duplicate sites identified before proposing an extraction
- [ ] Report ranks findings by impact × ease, omits checks with no real hits
- [ ] No code modified unless the user explicitly asked for fixes to be applied
- [ ] If fixes were applied: routed through the project's mandated subagent, and verified via a running dev server + console check
