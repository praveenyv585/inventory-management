<template>
  <aside class="sidebar" :class="{ collapsed, 'mobile-open': mobileOpen }">
    <!-- Brand -->
    <div class="sidebar-brand">
      <span class="brand-icon">{{ brandInitial }}</span>
      <span class="brand-name" v-if="!collapsed">{{ appName }}</span>
    </div>

    <!-- Nav items -->
    <nav class="sidebar-nav">
      <div v-for="item in navItems" :key="item.path" class="nav-item-wrapper">
        <router-link
          :to="item.path"
          class="nav-item"
          :class="{ active: isActive(item) }"
          @click="closeMobile"
          @mouseenter="collapsed ? showTooltip($event, item.label) : null"
          @mouseleave="hideTooltip"
        >
          <span class="nav-icon">{{ item.icon }}</span>
          <span class="nav-label" v-if="!collapsed">{{ item.label }}</span>
        </router-link>
      </div>
    </nav>

    <!-- Tooltip portal: renders outside sidebar to escape overflow:hidden clipping -->
    <Teleport to="body">
      <div
        v-if="tooltip.visible"
        class="sidebar-tooltip-portal"
        :style="{ top: tooltip.top + 'px', left: tooltip.left + 'px' }"
      >
        {{ tooltip.label }}
      </div>
    </Teleport>

    <!-- Footer: collapse toggle (desktop only) -->
    <div class="sidebar-footer">
      <button
        class="collapse-btn"
        @click="toggleCollapse"
        :title="collapsed ? 'Expand sidebar' : 'Collapse sidebar'"
      >
        <span class="collapse-icon">{{ collapsed ? '▶' : '◀' }}</span>
        <span class="collapse-label" v-if="!collapsed">Collapse</span>
      </button>
    </div>
  </aside>

  <!-- Mobile overlay backdrop -->
  <div v-if="mobileOpen" class="sidebar-backdrop" @click="closeMobile" />
</template>

<script>
  import { ref, computed, onMounted, onUnmounted } from 'vue'
  import { useRoute } from 'vue-router'

  const STORAGE_KEY = 'sidebar-collapsed'

  export default {
    name: 'AppSidebar',
    props: {
      navItems: { type: Array, required: true },
      appName: { type: String, default: '' },
    },
    emits: ['collapse-change'],
    setup(props, { emit }) {
      const route = useRoute()

      // Restore persisted collapsed state (default false)
      const stored = localStorage.getItem(STORAGE_KEY)
      const collapsed = ref(stored === 'true')

      // Mobile overlay state — driven by screen width
      const mobileOpen = ref(false)
      const isMobile = ref(window.innerWidth < 768)

      const brandInitial = computed(() => (props.appName ? props.appName[0].toUpperCase() : '●'))

      const isActive = (item) => {
        if (item.path === '/') return route.path === '/'
        return route.path.startsWith(item.path)
      }

      const toggleCollapse = () => {
        collapsed.value = !collapsed.value
        localStorage.setItem(STORAGE_KEY, String(collapsed.value))
        emit('collapse-change', collapsed.value)
      }

      const closeMobile = () => {
        mobileOpen.value = false
      }

      const onResize = () => {
        const nowMobile = window.innerWidth < 768
        if (nowMobile !== isMobile.value) {
          isMobile.value = nowMobile
          // Auto-collapse when window shrinks below breakpoint
          if (nowMobile && !collapsed.value) {
            collapsed.value = true
          }
        }
      }

      onMounted(() => {
        // Auto-collapse on initial load if screen is small
        if (window.innerWidth < 768) {
          collapsed.value = true
        }
        window.addEventListener('resize', onResize)
      })

      onUnmounted(() => {
        window.removeEventListener('resize', onResize)
      })

      const tooltip = ref({ visible: false, label: '', top: 0, left: 0 })

      const showTooltip = (event, label) => {
        const rect = event.currentTarget.getBoundingClientRect()
        tooltip.value = {
          visible: true,
          label,
          top: rect.top + rect.height / 2,
          left: rect.right + 10,
        }
      }

      const hideTooltip = () => {
        tooltip.value.visible = false
      }

      return {
        collapsed,
        mobileOpen,
        brandInitial,
        isActive,
        toggleCollapse,
        closeMobile,
        tooltip,
        showTooltip,
        hideTooltip,
      }
    },
  }
</script>

<style scoped>
  /* ── Sidebar shell ───────────────────────────────── */
  .sidebar {
    width: 220px;
    min-height: 100vh;
    background: #0f172a;
    border-right: 1px solid #1e293b;
    display: flex;
    flex-direction: column;
    transition: width 0.22s cubic-bezier(0.4, 0, 0.2, 1);
    flex-shrink: 0;
    position: sticky;
    top: 0;
    height: 100vh;
    overflow: hidden;
    z-index: 100;
  }

  .sidebar.collapsed {
    width: 56px;
  }

  /* ── Brand ───────────────────────────────────────── */
  .sidebar-brand {
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 20px 12px;
    border-bottom: 1px solid #1e293b;
    min-height: 64px;
    flex-shrink: 0;
    overflow: hidden;
  }

  .collapsed .sidebar-brand {
    justify-content: center;
    padding: 20px 0;
  }

  .brand-icon {
    width: 32px;
    height: 32px;
    background: #2563eb;
    color: #fff;
    border-radius: 8px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-weight: 800;
    font-size: 0.875rem;
    flex-shrink: 0;
  }

  .brand-name {
    font-size: 0.875rem;
    font-weight: 700;
    color: #ffffff;
    white-space: nowrap;
    letter-spacing: -0.01em;
    opacity: 1;
    transition: opacity 0.15s ease;
  }

  /* ── Nav ─────────────────────────────────────────── */
  .sidebar-nav {
    flex: 1;
    padding: 12px 8px;
    display: flex;
    flex-direction: column;
    gap: 2px;
    overflow-y: auto;
    overflow-x: hidden;
  }

  .collapsed .sidebar-nav {
    padding: 12px 4px;
    align-items: center;
  }

  .sidebar-nav::-webkit-scrollbar {
    width: 4px;
  }
  .sidebar-nav::-webkit-scrollbar-track {
    background: transparent;
  }
  .sidebar-nav::-webkit-scrollbar-thumb {
    background: #1e293b;
    border-radius: 2px;
  }

  /* Nav item wrapper — needed for tooltip positioning */
  .nav-item-wrapper {
    position: relative;
    width: 100%;
  }

  .nav-item-wrapper.collapsed {
    width: 40px;
  }

  .nav-item {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 8px 12px;
    border-radius: 8px;
    color: #94a3b8;
    text-decoration: none;
    font-size: 0.875rem;
    font-weight: 500;
    white-space: nowrap;
    transition:
      background 0.12s ease,
      color 0.12s ease;
    min-height: 36px;
    width: 100%;
  }

  .collapsed .nav-item {
    padding: 8px;
    justify-content: center;
    width: 40px;
  }

  .nav-item:hover {
    background: #1e293b;
    color: #f1f5f9;
  }
  .nav-item.active {
    background: #1e3a5f;
    color: #ffffff;
  }

  .nav-icon {
    font-size: 1.05rem;
    width: 20px;
    text-align: center;
    flex-shrink: 0;
    line-height: 1;
  }

  .nav-label {
    overflow: hidden;
    text-overflow: ellipsis;
    flex: 1;
  }

  /* Tooltip is now a <Teleport to="body"> portal — styles live in App.vue global */

  /* ── Footer / collapse toggle ────────────────────── */
  .sidebar-footer {
    padding: 12px 8px;
    border-top: 1px solid #1e293b;
    flex-shrink: 0;
  }

  .collapsed .sidebar-footer {
    padding: 12px 4px;
    display: flex;
    justify-content: center;
  }

  .collapse-btn {
    width: 100%;
    background: none;
    border: none;
    color: #64748b;
    cursor: pointer;
    padding: 7px 12px;
    border-radius: 8px;
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 0.8rem;
    font-weight: 500;
    transition:
      background 0.12s,
      color 0.12s;
  }

  .collapsed .collapse-btn {
    width: 40px;
    padding: 7px 0;
    justify-content: center;
  }

  .collapse-btn:hover {
    background: #1e293b;
    color: #f1f5f9;
  }

  .collapse-icon {
    font-size: 0.65rem;
    width: 20px;
    text-align: center;
    flex-shrink: 0;
  }

  .collapse-label {
    white-space: nowrap;
  }

  /* ── Mobile overlay backdrop ─────────────────────── */
  .sidebar-backdrop {
    display: none;
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.5);
    z-index: 99;
  }

  /* ── Responsive ──────────────────────────────────── */
  @media (max-width: 767px) {
    .sidebar {
      position: fixed;
      left: 0;
      top: 0;
      height: 100vh;
      /* On mobile the sidebar slides in/out as an overlay */
      transform: translateX(-100%);
      transition:
        transform 0.22s cubic-bezier(0.4, 0, 0.2, 1),
        width 0.22s cubic-bezier(0.4, 0, 0.2, 1);
    }

    .sidebar.collapsed {
      /* Collapsed on mobile = icon-only strip visible */
      width: 56px;
      transform: translateX(0);
    }

    .sidebar.mobile-open {
      width: 220px;
      transform: translateX(0);
    }

    .sidebar-backdrop {
      display: block;
    }
  }
</style>
