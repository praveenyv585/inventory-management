---
name: vue-saas-redesign
description: Redesigns a Vue 3 application's UI into a modern SaaS-style interface — replaces the top nav bar with a vertical left sidebar, applies a consistent design system (spacing, typography, color tokens), and polishes the overall look without touching routes, business logic, or composables.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# /vue-saas-redesign — Vue 3 SaaS UI Redesign

This skill transforms a Vue 3 app with a horizontal top-nav layout into a modern SaaS-style interface with a vertical left sidebar. It replaces only the shell (App.vue, global styles, layout wrapper) and creates a new Sidebar component. It does **not** touch routes, composables, API clients, view logic, or backend code.

The output looks like Linear, Vercel, or Notion: a narrow sidebar on the left for navigation, a slim top bar across the right pane for context actions (user menu, language, breadcrumb), and a clean content area with consistent spacing.

**When invoked, work through all seven phases in order.** Read before you edit. Do not skip phases.

---

## Phase 1 — Audit the existing app

Before writing a single line, understand what you're working with.

### 1a. Map the shell files

Read these files in full:
- `src/App.vue` — the top-level layout, current nav, any modals/overlays
- `src/main.js` — router setup, route names and paths
- Any existing global stylesheet (`src/assets/main.css`, `src/index.css`, or the `<style>` block in App.vue)

Extract and note:
- Every `<router-link>` and its `to` path — these become sidebar items
- Every component imported into App.vue (FilterBar, ProfileMenu, LanguageSwitcher, modals, etc.)
- Every globally-scoped CSS class that other views depend on (`.card`, `.badge`, `.loading`, `.error`, `.btn-*`, `.page-header`, table rules, etc.)

### 1b. Map the views

Run `Glob("src/views/**/*.vue")` to list all views. For each, note its route path from main.js and what page-level wrapper classes it uses (`.page-header`, inner `.card` blocks, etc.).

### 1c. Identify design tokens already in use

Grep for hard-coded hex colors in App.vue and the most complex view. List the palette — you will replace it with a unified token set in Phase 2.

---

## Phase 2 — Define the design system

Apply this token set. Write these as CSS custom properties on `:root` inside the global style block of App.vue. Do not scatter them elsewhere.

### Color tokens

```css
:root {
  /* Brand */
  --color-accent:        #2563eb;   /* primary CTA, active states */
  --color-accent-hover:  #1d4ed8;
  --color-accent-bg:     #eff6ff;   /* active nav item background */
  --color-accent-text:   #1e40af;   /* active nav item text */

  /* Sidebar */
  --sidebar-bg:          #0f172a;   /* dark navy sidebar */
  --sidebar-text:        #94a3b8;   /* inactive nav item */
  --sidebar-text-hover:  #f1f5f9;
  --sidebar-text-active: #ffffff;
  --sidebar-item-hover:  #1e293b;
  --sidebar-item-active: #1e3a5f;
  --sidebar-border:      #1e293b;
  --sidebar-width:       220px;
  --sidebar-collapsed:   56px;

  /* Surface */
  --surface-page:        #f8fafc;   /* overall page background */
  --surface-card:        #ffffff;
  --surface-card-hover:  #f8fafc;

  /* Borders */
  --border-subtle:       #e2e8f0;
  --border-default:      #cbd5e1;

  /* Text */
  --text-primary:        #0f172a;
  --text-secondary:      #475569;
  --text-muted:          #94a3b8;

  /* Status (keep existing values, just reference via tokens) */
  --status-success-bg:   #d1fae5;
  --status-success-text: #065f46;
  --status-warning-bg:   #fed7aa;
  --status-warning-text: #92400e;
  --status-danger-bg:    #fecaca;
  --status-danger-text:  #991b1b;
  --status-info-bg:      #dbeafe;
  --status-info-text:    #1e40af;

  /* Spacing scale (4px base) */
  --space-1:  4px;
  --space-2:  8px;
  --space-3:  12px;
  --space-4:  16px;
  --space-5:  20px;
  --space-6:  24px;
  --space-8:  32px;
  --space-10: 40px;

  /* Typography */
  --font-sans: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  --font-mono: 'Cascadia Code', 'Fira Code', Consolas, monospace;
  --text-xs:   0.75rem;
  --text-sm:   0.875rem;
  --text-base: 1rem;
  --text-lg:   1.125rem;
  --text-xl:   1.25rem;
  --text-2xl:  1.5rem;
  --text-3xl:  1.875rem;

  /* Radii */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;

  /* Shadows */
  --shadow-sm: 0 1px 2px 0 rgba(0,0,0,.05);
  --shadow-md: 0 4px 6px -1px rgba(0,0,0,.08), 0 2px 4px -1px rgba(0,0,0,.04);
  --shadow-lg: 0 10px 15px -3px rgba(0,0,0,.1), 0 4px 6px -2px rgba(0,0,0,.05);
}
```

### When to deviate

- If the app already has a light-themed sidebar or the user specifies `light sidebar`, swap `--sidebar-bg` to `#ffffff`, set `--sidebar-text` to `#475569`, `--sidebar-text-active` to `#0f172a`, `--sidebar-item-active` to `#eff6ff`, and `--sidebar-border` to `#e2e8f0`. All other tokens stay the same.
- Do not invent new colors. If a view needs a color not in the token set, pick the nearest token.

---

## Phase 3 — Create `src/components/AppSidebar.vue`

Create this file from scratch. It is the new primary navigation shell.

### Structure

The sidebar has three vertical zones:
1. **Brand area** — logo/app name at the top (matches what was in the old top nav)
2. **Nav items** — router-links for every route previously in the top nav
3. **Footer area** — user avatar + name at the bottom (optional: collapse toggle)

### Template skeleton

```vue
<template>
  <aside class="sidebar" :class="{ collapsed }">
    <!-- Brand -->
    <div class="sidebar-brand">
      <span class="brand-icon">{{ brandInitial }}</span>
      <span class="brand-name" v-if="!collapsed">{{ appName }}</span>
    </div>

    <!-- Nav items -->
    <nav class="sidebar-nav">
      <router-link
        v-for="item in navItems"
        :key="item.path"
        :to="item.path"
        class="nav-item"
        :class="{ active: isActive(item) }"
        :title="collapsed ? item.label : undefined"
      >
        <span class="nav-icon">{{ item.icon }}</span>
        <span class="nav-label" v-if="!collapsed">{{ item.label }}</span>
      </router-link>
    </nav>

    <!-- Footer -->
    <div class="sidebar-footer">
      <button class="collapse-btn" @click="collapsed = !collapsed" :title="collapsed ? 'Expand' : 'Collapse'">
        <span>{{ collapsed ? '→' : '←' }}</span>
      </button>
    </div>
  </aside>
</template>
```

### Script

```vue
<script>
import { ref, computed } from 'vue'
import { useRoute } from 'vue-router'

export default {
  name: 'AppSidebar',
  props: {
    navItems: { type: Array, required: true },  // [{ path, label, icon }]
    appName:  { type: String, default: '' }
  },
  setup(props) {
    const route = useRoute()
    const collapsed = ref(false)

    const brandInitial = computed(() =>
      props.appName ? props.appName[0].toUpperCase() : '●'
    )

    const isActive = (item) => {
      if (item.path === '/') return route.path === '/'
      return route.path.startsWith(item.path)
    }

    return { collapsed, brandInitial, isActive }
  }
}
</script>
```

### Nav items array

Derive this from the routes you found in Phase 1. Use text symbols for icons (no icon library needed):

| Route | Icon | Notes |
|-------|------|-------|
| `/` (overview/dashboard) | `◉` | |
| `/inventory` | `⬡` | |
| `/orders` | `⬜` | |
| `/spending` or `/finance` | `◈` | |
| `/demand` | `△` | |
| `/reports` | `▤` | |
| `/restocking` | `↺` | |
| `/backlog` | `⊟` | |
| Any unknown route | `◦` | fallback |

If the app already uses an icon library (heroicons, lucide, etc.), keep using it. Only use text symbols if no icon library is installed.

### Scoped styles

```css
.sidebar {
  width: var(--sidebar-width);
  min-height: 100vh;
  background: var(--sidebar-bg);
  border-right: 1px solid var(--sidebar-border);
  display: flex;
  flex-direction: column;
  transition: width 0.2s ease;
  flex-shrink: 0;
  position: sticky;
  top: 0;
  overflow: hidden;
}

.sidebar.collapsed { width: var(--sidebar-collapsed); }

/* Brand */
.sidebar-brand {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-5) var(--space-4);
  border-bottom: 1px solid var(--sidebar-border);
  min-height: 64px;
}
.brand-icon {
  width: 32px; height: 32px;
  background: var(--color-accent);
  color: #fff;
  border-radius: var(--radius-md);
  display: flex; align-items: center; justify-content: center;
  font-weight: 800; font-size: var(--text-sm);
  flex-shrink: 0;
}
.brand-name {
  font-size: var(--text-sm);
  font-weight: 700;
  color: var(--sidebar-text-active);
  white-space: nowrap;
  letter-spacing: -0.01em;
}

/* Nav */
.sidebar-nav {
  flex: 1;
  padding: var(--space-3) var(--space-2);
  display: flex;
  flex-direction: column;
  gap: 2px;
  overflow-y: auto;
  overflow-x: hidden;
}
.nav-item {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-3);
  border-radius: var(--radius-md);
  color: var(--sidebar-text);
  text-decoration: none;
  font-size: var(--text-sm);
  font-weight: 500;
  white-space: nowrap;
  transition: all 0.15s ease;
  min-height: 36px;
}
.nav-item:hover {
  background: var(--sidebar-item-hover);
  color: var(--sidebar-text-hover);
}
.nav-item.active {
  background: var(--sidebar-item-active);
  color: var(--sidebar-text-active);
}
.nav-icon {
  font-size: 1rem;
  width: 20px;
  text-align: center;
  flex-shrink: 0;
}
.nav-label { overflow: hidden; text-overflow: ellipsis; }

/* Footer */
.sidebar-footer {
  padding: var(--space-3) var(--space-2);
  border-top: 1px solid var(--sidebar-border);
}
.collapse-btn {
  width: 100%;
  background: none;
  border: none;
  color: var(--sidebar-text);
  cursor: pointer;
  padding: var(--space-2) var(--space-3);
  border-radius: var(--radius-md);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: var(--text-sm);
  transition: all 0.15s;
}
.collapse-btn:hover {
  background: var(--sidebar-item-hover);
  color: var(--sidebar-text-hover);
}
```

---

## Phase 4 — Rewrite `App.vue`

This is the most impactful change. The goal: replace the horizontal `<header class="top-nav">` with a two-column layout (`<aside>` + `<div class="app-body">`).

### New layout structure

```
┌────────────────────────────────────────────────────┐
│  .app (display: flex; flex-direction: row)         │
│  ┌──────────┐  ┌──────────────────────────────────┐│
│  │          │  │  .app-body (flex: 1)              ││
│  │AppSidebar│  │  ┌────────────────────────────────┐│
│  │  220px   │  │  │  .topbar (sticky, 52px)        ││
│  │  sticky  │  │  └────────────────────────────────┘│
│  │          │  │  ┌────────────────────────────────┐│
│  │          │  │  │  FilterBar (if present)        ││
│  │          │  │  └────────────────────────────────┘│
│  │          │  │  ┌────────────────────────────────┐│
│  │          │  │  │  .main-content                 ││
│  │          │  │  │  <router-view />               ││
│  │          │  │  └────────────────────────────────┘│
│  └──────────┘  └──────────────────────────────────┘│
└────────────────────────────────────────────────────┘
```

### App.vue template

```vue
<template>
  <div class="app">
    <AppSidebar :nav-items="navItems" :app-name="t('nav.companyName')" />

    <div class="app-body">
      <!-- Slim top bar: right-side utilities only -->
      <header class="topbar">
        <div class="topbar-left">
          <!-- Breadcrumb or page title slot -->
          <span class="topbar-route">{{ currentPageTitle }}</span>
        </div>
        <div class="topbar-right">
          <!-- Preserve any components that were in the old top nav's right side -->
          <!-- e.g.: LanguageSwitcher, ProfileMenu -->
          <LanguageSwitcher v-if="hasLanguageSwitcher" />
          <ProfileMenu
            v-if="hasProfileMenu"
            @show-profile-details="showProfileDetails = true"
            @show-tasks="showTasks = true"
          />
        </div>
      </header>

      <!-- Global filter bar (if the app has one) -->
      <FilterBar v-if="hasFilterBar" />

      <!-- Page content -->
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <!-- Preserve all modals exactly as they were -->
  </div>
</template>
```

**Important:** Do not remove any components that were in the original App.vue. Move them to the appropriate zone:
- Nav links → `AppSidebar` navItems prop
- Logo/brand → `AppSidebar` appName prop
- Language switcher, profile menu → `topbar-right`
- Filter bar → between topbar and main-content
- All modals → keep at bottom of template, exactly as before

### navItems array in `setup()`

Build navItems from the routes you mapped in Phase 1, calling the i18n `t()` function for labels:

```js
const navItems = [
  { path: '/',           label: t('nav.overview'),       icon: '◉' },
  { path: '/inventory',  label: t('nav.inventory'),      icon: '⬡' },
  { path: '/orders',     label: t('nav.orders'),         icon: '⬜' },
  // ... one entry per route
]
```

### currentPageTitle computed

```js
const { t } = useI18n()
const route = useRoute()

const PAGE_TITLES = {
  '/':           () => t('nav.overview'),
  '/inventory':  () => t('nav.inventory'),
  // ... map every route path to its label
}

const currentPageTitle = computed(() =>
  (PAGE_TITLES[route.path] || (() => ''))()
)
```

### App.vue global styles

Replace the entire `<style>` block (not scoped) with the following. Preserve every rule that other views depend on (`.card`, `.badge`, `.loading`, `.error`, table rules, `.btn-*`, `.stat-card`, `.page-header`). Rewrite only the layout-level and nav rules.

```css
/* ── Reset & base ───────────────────────────────── */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: var(--font-sans);
  background: var(--surface-page);
  color: var(--text-primary);
  -webkit-font-smoothing: antialiased;
  overflow-x: hidden;
}

/* ── App shell ──────────────────────────────────── */
.app {
  display: flex;
  min-height: 100vh;
}

.app-body {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;          /* prevents flex overflow */
  min-height: 100vh;
}

/* ── Topbar ─────────────────────────────────────── */
.topbar {
  height: 52px;
  background: var(--surface-card);
  border-bottom: 1px solid var(--border-subtle);
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 var(--space-6);
  position: sticky;
  top: 0;
  z-index: 90;
  flex-shrink: 0;
}
.topbar-left { display: flex; align-items: center; gap: var(--space-3); }
.topbar-route {
  font-size: var(--text-sm);
  font-weight: 600;
  color: var(--text-secondary);
}
.topbar-right { display: flex; align-items: center; gap: var(--space-3); }

/* ── Main content ───────────────────────────────── */
.main-content {
  flex: 1;
  padding: var(--space-6);
  max-width: 1400px;
  width: 100%;
}

/* ── Page header ────────────────────────────────── */
.page-header { margin-bottom: var(--space-6); }
.page-header h2 {
  font-size: var(--text-2xl);
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.025em;
  margin-bottom: var(--space-1);
}
.page-header p { color: var(--text-secondary); font-size: var(--text-sm); }

/* ── Cards ──────────────────────────────────────── */
.card {
  background: var(--surface-card);
  border-radius: var(--radius-lg);
  border: 1px solid var(--border-subtle);
  padding: var(--space-5);
  margin-bottom: var(--space-5);
  box-shadow: var(--shadow-sm);
}
.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--space-4);
  padding-bottom: var(--space-4);
  border-bottom: 1px solid var(--border-subtle);
}
.card-title {
  font-size: var(--text-base);
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.01em;
}

/* ── Stat cards ─────────────────────────────────── */
.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: var(--space-4);
  margin-bottom: var(--space-5);
}
.stat-card {
  background: var(--surface-card);
  padding: var(--space-5);
  border-radius: var(--radius-lg);
  border: 1px solid var(--border-subtle);
  box-shadow: var(--shadow-sm);
  transition: box-shadow 0.15s;
}
.stat-card:hover { box-shadow: var(--shadow-md); }
.stat-label {
  font-size: var(--text-xs);
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: var(--text-muted);
  margin-bottom: var(--space-2);
}
.stat-value {
  font-size: 2rem;
  font-weight: 800;
  color: var(--text-primary);
  letter-spacing: -0.03em;
  line-height: 1;
}
.stat-card.success .stat-value { color: #059669; }
.stat-card.warning .stat-value { color: #d97706; }
.stat-card.danger  .stat-value { color: #dc2626; }
.stat-card.info    .stat-value { color: var(--color-accent); }

/* ── Tables ─────────────────────────────────────── */
.table-container { overflow-x: auto; }
table { width: 100%; border-collapse: collapse; }
thead {
  background: var(--surface-page);
  border-top: 1px solid var(--border-subtle);
  border-bottom: 1px solid var(--border-subtle);
}
th {
  text-align: left;
  padding: var(--space-2) var(--space-4);
  font-size: var(--text-xs);
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: var(--text-muted);
}
td {
  padding: var(--space-3) var(--space-4);
  border-top: 1px solid #f1f5f9;
  color: var(--text-secondary);
  font-size: var(--text-sm);
}
tbody tr { transition: background-color 0.1s; }
tbody tr:hover { background: var(--surface-card-hover); }

/* ── Badges ─────────────────────────────────────── */
.badge {
  display: inline-block;
  padding: 0.2rem 0.6rem;
  border-radius: 5px;
  font-size: var(--text-xs);
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}
.badge.success   { background: var(--status-success-bg); color: var(--status-success-text); }
.badge.warning   { background: var(--status-warning-bg); color: var(--status-warning-text); }
.badge.danger    { background: var(--status-danger-bg);  color: var(--status-danger-text);  }
.badge.info      { background: var(--status-info-bg);    color: var(--status-info-text);    }
.badge.increasing { background: var(--status-success-bg); color: var(--status-success-text); }
.badge.decreasing { background: var(--status-danger-bg);  color: var(--status-danger-text);  }
.badge.stable     { background: #e0e7ff; color: #3730a3; }

/* ── State: loading / error ─────────────────────── */
.loading {
  text-align: center;
  padding: var(--space-10);
  color: var(--text-muted);
  font-size: var(--text-sm);
}
.error {
  background: var(--status-danger-bg);
  border: 1px solid #fca5a5;
  color: var(--status-danger-text);
  padding: var(--space-4);
  border-radius: var(--radius-md);
  margin: var(--space-4) 0;
  font-size: var(--text-sm);
}

/* ── Responsive ─────────────────────────────────── */
@media (max-width: 768px) {
  .app { flex-direction: column; }
  .sidebar { width: 100% !important; min-height: auto; flex-direction: row; }
  /* On mobile the sidebar collapses to a top strip — you can expand this */
}
```

---

## Phase 5 — Remove old nav styles

After rewriting App.vue's style block, grep the file for any remaining references to:
- `.top-nav`, `.nav-container`, `.nav-tabs`, `.logo`, `.subtitle`

Remove them entirely — they are dead code once the sidebar is in place.

---

## Phase 6 — View-level adjustments

Open each view file and apply these fixes:

### Remove redundant top padding

Each view currently assumes its content starts below a 70px top nav. With a sticky topbar of 52px now handled by the shell, views should **not** add their own top padding. If you see `padding-top: 70px` or similar on the root element, remove it.

### Update filter bar (if present)

If a `FilterBar.vue` component exists and was previously inside the sticky top nav, it is now in the `app-body` flow below the topbar. Update its container style:

```css
/* FilterBar.vue: change the wrapper to match the new shell */
.filter-bar {
  background: var(--surface-card);
  border-bottom: 1px solid var(--border-subtle);
  padding: var(--space-3) var(--space-6);
  display: flex;
  align-items: center;
  gap: var(--space-4);
  flex-shrink: 0;
}
```

### Ensure `.page-header` margin is correct

Each view's `.page-header` block should have `margin-bottom: var(--space-6)` and nothing more. Remove any explicit `margin-top` on `.page-header` inside views — the `main-content` padding handles the top gap.

---

## Phase 7 — Responsive and edge cases

### Sidebar collapse on narrow screens

The sidebar collapse toggle (the `←` / `→` button in Phase 3) handles wide-screen collapsing. On screens narrower than 768px, the sidebar becomes a horizontal strip at the top — the default responsive rule in Phase 4's styles handles this.

If the app needs a mobile hamburger menu, add a `showMobile` ref to AppSidebar and emit it to App.vue through the `@toggle-mobile` event. This is optional and only needed if mobile is a stated requirement.

### `min-width: 0` is mandatory

Any `flex` child that contains a table or wide content **must** have `min-width: 0` set on it, otherwise the content overflows the flex parent. The `.app-body` rule in Phase 4 already includes this. If individual views still overflow, add `min-width: 0` to their root element's scoped style.

### Scrollbar appearance

Add this to the global style block to polish the scrollbar in the sidebar nav:

```css
.sidebar-nav::-webkit-scrollbar { width: 4px; }
.sidebar-nav::-webkit-scrollbar-track { background: transparent; }
.sidebar-nav::-webkit-scrollbar-thumb {
  background: var(--sidebar-border);
  border-radius: 2px;
}
```

### Z-index layer order

Keep these z-index values consistent:
- Sidebar: no explicit z-index (static/sticky, not overlapping)
- Topbar: `z-index: 90`
- FilterBar: `z-index: 80`
- Modals / overlays: `z-index: 200+`
- Dropdown menus inside views: `z-index: 10` (scoped)

---

## Implementation notes

- **Read every file before editing it.** The `Edit` tool requires a prior `Read`. Never guess at file contents.
- **Don't touch routes, composables, or API clients.** The skill scope is the shell and global styles only. Business logic, data fetching, and state management are out of scope.
- **Don't add icon libraries.** Use text symbols or Unicode characters. If the app already has an icon library, use it consistently.
- **Preserve all existing modal and overlay components** exactly — just move them to the appropriate place in the new App.vue template.
- **The global style block in App.vue is not scoped.** All rules there affect every view. Before replacing a rule, confirm no view depends on the old class name. If a view has its own scoped version of the same class, the global one can safely be removed or simplified.
- **Test the active state logic in AppSidebar.** Routes like `/` need an exact match (`route.path === '/'`). All others should use `startsWith` so sub-routes highlight the correct nav item.
- **i18n keys for nav labels.** The `navItems` array should call `t()` reactively. If the app uses an `useI18n()` composable, call it in `App.vue`'s setup and pass translated strings as the `label` field. Don't pass raw `t` calls to AppSidebar as a prop — pass the computed string.
- **After all edits, re-read App.vue** to confirm the `:root` token block is present, the `.top-nav` / `.nav-tabs` rules are gone, AppSidebar is imported and used, and all modals are still in the template.
