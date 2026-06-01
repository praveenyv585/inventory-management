---
name: vue-analyzer
description: Analyzes Vue 3 component structure and suggests concrete optimizations for performance (computed properties, v-if/v-show, re-renders, async loading) and code reuse (extractable composables, shared components, duplicated logic).
user-invocable: true
allowed-tools:
  - Read
  - Glob
  - Grep
  - Write
  - Edit
---

# /vue-analyzer — Vue 3 Component Optimizer

This skill audits Vue 3 components in the current project and produces a prioritized report of performance issues and code-reuse opportunities, with concrete fix suggestions and code examples.

**Work through all four phases in order.** Do not skip phases. Read before you report.

---

## Phase 1 — Discover components

### 1a. Enumerate all Vue files

Glob for every `.vue` file under `src/`:

```
src/**/*.vue
```

Group them by type:
- **Views** — files in `src/views/`
- **Components** — files in `src/components/`
- **App shell** — `src/App.vue`

### 1b. Enumerate composables

Glob for every `.js` / `.ts` file under `src/composables/` (or `src/use*/`). These are the existing reuse primitives — note what each one does so you can spot duplication.

### 1c. Read the API client

Read `src/api.js` (or equivalent). Note every exported function — these are the data-fetching primitives. Duplicated fetch calls across components are a code-reuse smell.

---

## Phase 2 — Analyze each component

Read every `.vue` file discovered in Phase 1. For each file, check all of the following categories. Record every finding with: **file**, **line range**, **category**, **severity** (High / Medium / Low), and a one-sentence description.

### 2a. Performance — Computed vs inline expressions

**Problem**: Heavy or repeated expressions written directly in the template re-run on every render.

Look for:
- Template expressions with `.filter()`, `.map()`, `.reduce()`, `.sort()` — these should be `computed`
- The same expression used more than once in the same template
- Method calls in `:class`, `:style`, or `v-if` bindings that don't depend on user interaction

**Severity**: High if the array is large or the operation is O(n²); Medium otherwise.

### 2b. Performance — `v-if` vs `v-show`

**Problem**: `v-if` destroys and recreates DOM on every toggle; `v-show` just toggles CSS `display`.

Look for:
- `v-if` on elements that toggle frequently (modals, dropdowns, tabs, loading spinners)
- `v-show` on elements that are almost never shown (conditional error states, empty-state placeholders)

**Severity**: Medium. Use `v-show` for frequent toggles, `v-if` for infrequent ones.

### 2c. Performance — Watchers that should be computed

**Problem**: A `watch` that only derives a value from reactive state is better expressed as a `computed`.

Look for:
- `watch(source, (val) => { derivedRef.value = transform(val) })` — this is a computed in disguise
- Watchers with no side effects (no API calls, no DOM manipulation, no emits)

**Severity**: Medium.

### 2d. Performance — Missing `key` in `v-for`

**Problem**: Without a stable `key`, Vue cannot efficiently diff list updates and will re-render entire rows.

Look for:
- `v-for` without `:key`
- `:key="index"` — index keys break diffing when items are reordered or deleted

**Severity**: High.

### 2e. Performance — Heavy components not lazy-loaded

**Problem**: Large components imported statically increase the initial bundle and slow first paint.

Look for:
- Static `import` of components that are only shown after user interaction (modals, drawers, detail panels, chart components)
- Components that import heavy libraries (chart libs, rich-text editors, date pickers)

**Fix pattern**:
```js
// Before
import HeavyChart from './HeavyChart.vue'

// After
import { defineAsyncComponent } from 'vue'
const HeavyChart = defineAsyncComponent(() => import('./HeavyChart.vue'))
```

**Severity**: Medium for large components; Low for small ones.

### 2f. Performance — `shallowRef` / `shallowReactive` for large objects

**Problem**: `ref()` and `reactive()` deeply observe every nested property. For large read-only datasets this wastes memory and slows reactivity.

Look for:
- `ref([...])` or `reactive({...})` holding large arrays or deeply nested objects that are replaced wholesale (not mutated property by property)
- API response data stored in `ref` that is only ever reassigned, never mutated in place

**Fix**: Use `shallowRef` when you replace the whole value; use `readonly` when the data never changes after load.

**Severity**: Low unless the dataset is large (hundreds of items).

### 2g. Code reuse — Duplicated template blocks

**Problem**: The same HTML structure repeated in multiple components is a candidate for a shared component.

Look for:
- Card/panel wrappers with the same structure across views
- Table headers or row layouts repeated verbatim
- Stat/metric display patterns (label + value + trend) used in more than one place
- Loading skeletons or empty-state blocks copy-pasted across views

**Severity**: Medium. Note all files where the duplication appears.

### 2h. Code reuse — Logic that belongs in a composable

**Problem**: `setup()` functions that contain data-fetching, filtering, or formatting logic repeated across components.

Look for:
- The same `onMounted(() => { fetchData() })` + `ref` + `computed` pattern in multiple components
- Filter/search logic (same `computed` filtering the same kind of array) in more than one view
- Date formatting, currency formatting, or status-label mapping repeated inline

**Fix pattern**:
```js
// Extract to src/composables/useInventory.js
export function useInventory(filters) {
  const items = shallowRef([])
  const filtered = computed(() => ...)
  onMounted(async () => { items.value = await fetchInventory(filters) })
  return { items, filtered }
}
```

**Severity**: High if duplicated in 3+ components; Medium for 2.

### 2i. Code reuse — Direct API calls outside `api.js`

**Problem**: `fetch()`/`axios` calls written directly in components instead of going through the central API client bypass error handling and make mocking harder.

Look for:
- `fetch(` or `axios.` inside `.vue` files
- `useRoute` + manual URL construction inside components

**Severity**: High.

---

## Phase 3 — Cross-component analysis

After reading all components, look for patterns that only appear when comparing files side by side.

### 3a. Composable extraction candidates

Group findings from 2h by the logic pattern. If the same fetch+filter+compute pattern appears in 2+ views, name a composable that would eliminate the duplication and list which files it would replace.

### 3b. Shared component candidates

Group findings from 2g. For each duplicated block, propose a component name, its props interface, and which files would use it.

### 3c. Prop-drilling chains

Look for data passed as props more than 2 levels deep (parent → child → grandchild). These are candidates for `provide`/`inject` or a shared composable.

---

## Phase 4 — Report

Produce a structured Markdown report with the following sections. Print it to the terminal (do not write a file unless the user asks).

```
# Vue Component Analysis Report

## Summary
- Files analyzed: N
- Total findings: N (H high / M medium / L low)

## High Priority

### [CATEGORY] filename.vue:line
**Issue**: one sentence
**Fix**:
\`\`\`vue
// concrete before/after code snippet
\`\`\`

## Medium Priority
... (same format)

## Low Priority
... (same format)

## Reuse Opportunities

### Proposed composable: `useXxx`
Eliminates duplication in: ViewA.vue, ViewB.vue, ViewC.vue
Extracts: [description of the logic]

### Proposed component: `<XxxCard>`
Replaces duplicated block in: ComponentA.vue, ComponentB.vue
Props: { title: string, value: number, trend: string }

## Quick Wins (no-code changes)
- List of v-for key fixes and v-if/v-show swaps that are one-line changes
```

### Report rules
- Lead with High findings — these have the biggest impact.
- Every finding must include a file path and line range so the user can navigate directly.
- Every fix must include a concrete before/after code snippet, not just a description.
- Do not report findings you are not confident about. A short accurate report is better than a long speculative one.
- If a component is well-structured with no findings, say so explicitly — it's useful signal.
