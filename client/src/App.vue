<template>
  <div class="app">
    <AppSidebar
      :nav-items="navItems"
      :app-name="t('nav.companyName')"
      @collapse-change="sidebarCollapsed = $event"
    />

    <div class="app-body" :class="{ 'sidebar-collapsed': sidebarCollapsed }">
      <!-- Slim topbar: utilities only -->
      <header class="topbar">
        <div class="topbar-left">
          <span class="topbar-route">{{ currentPageTitle }}</span>
        </div>
        <div class="topbar-right">
          <LanguageSwitcher />
          <ProfileMenu
            @show-profile-details="showProfileDetails = true"
            @show-tasks="showTasks = true"
          />
        </div>
      </header>

      <FilterBar />

      <main class="main-content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { useRoute } from 'vue-router'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import AppSidebar from './components/AppSidebar.vue'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

export default {
  name: 'App',
  components: {
    AppSidebar,
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const route = useRoute()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])
    const sidebarCollapsed = ref(localStorage.getItem('sidebar-collapsed') === 'true')

    const navItems = computed(() => [
      { path: '/',           label: t('nav.overview'),       icon: '◉' },
      { path: '/inventory',  label: t('nav.inventory'),      icon: '⬡' },
      { path: '/orders',     label: t('nav.orders'),         icon: '⬜' },
      { path: '/spending',   label: t('nav.finance'),        icon: '◈' },
      { path: '/demand',     label: t('nav.demandForecast'), icon: '△' },
      { path: '/reports',    label: 'Reports',               icon: '▤' },
      { path: '/restocking', label: t('nav.restocking'),     icon: '↺' },
    ])

    const pageTitleMap = computed(() => ({
      '/':           t('nav.overview'),
      '/inventory':  t('nav.inventory'),
      '/orders':     t('nav.orders'),
      '/spending':   t('nav.finance'),
      '/demand':     t('nav.demandForecast'),
      '/reports':    'Reports',
      '/restocking': t('nav.restocking'),
    }))

    const currentPageTitle = computed(() => pageTitleMap.value[route.path] || '')

    const tasks = computed(() => [...currentUser.value.tasks, ...apiTasks.value])

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)
        if (isMockTask) {
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) currentUser.value.tasks.splice(index, 1)
        } else {
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)
        if (mockTask) {
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) apiTasks.value[index] = updatedTask
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    onMounted(loadTasks)

    return {
      t,
      navItems,
      currentPageTitle,
      showProfileDetails,
      showTasks,
      sidebarCollapsed,
      tasks,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
/* ── Design tokens ───────────────────────────────── */
:root {
  --color-accent:        #2563eb;
  --color-accent-hover:  #1d4ed8;
  --color-accent-bg:     #eff6ff;
  --color-accent-text:   #1e40af;

  --sidebar-bg:          #0f172a;
  --sidebar-text:        #94a3b8;
  --sidebar-text-hover:  #f1f5f9;
  --sidebar-text-active: #ffffff;
  --sidebar-item-hover:  #1e293b;
  --sidebar-item-active: #1e3a5f;
  --sidebar-border:      #1e293b;
  --sidebar-width:       220px;
  --sidebar-collapsed:   56px;

  --surface-page:        #f8fafc;
  --surface-card:        #ffffff;
  --surface-card-hover:  #f8fafc;

  --border-subtle:       #e2e8f0;
  --border-default:      #cbd5e1;

  --text-primary:        #0f172a;
  --text-secondary:      #475569;
  --text-muted:          #94a3b8;

  --status-success-bg:   #d1fae5;
  --status-success-text: #065f46;
  --status-warning-bg:   #fed7aa;
  --status-warning-text: #92400e;
  --status-danger-bg:    #fecaca;
  --status-danger-text:  #991b1b;
  --status-info-bg:      #dbeafe;
  --status-info-text:    #1e40af;

  --space-1:  4px;   --space-2: 8px;   --space-3: 12px;
  --space-4:  16px;  --space-5: 20px;  --space-6: 24px;
  --space-8:  32px;  --space-10: 40px;

  --font-sans: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  --font-mono: 'Cascadia Code', 'Fira Code', Consolas, monospace;

  --radius-sm: 4px;  --radius-md: 8px;  --radius-lg: 12px;
  --shadow-sm: 0 1px 2px 0 rgba(0,0,0,.05);
  --shadow-md: 0 4px 6px -1px rgba(0,0,0,.08), 0 2px 4px -1px rgba(0,0,0,.04);
}

/* ── Reset ───────────────────────────────────────── */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: var(--font-sans);
  background: var(--surface-page);
  color: var(--text-primary);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  overflow-x: hidden;
}

/* ── App shell ───────────────────────────────────── */
.app {
  display: flex;
  min-height: 100vh;
}

.app-body {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
  min-height: 100vh;
  background: var(--surface-page);
}

/* ── Topbar ──────────────────────────────────────── */
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
  box-shadow: var(--shadow-sm);
}

.topbar-left { display: flex; align-items: center; gap: var(--space-3); }
.topbar-route {
  font-size: 0.875rem;
  font-weight: 600;
  color: var(--text-secondary);
  letter-spacing: -0.01em;
}
.topbar-right { display: flex; align-items: center; gap: var(--space-3); }

/* ── Main content ────────────────────────────────── */
.main-content {
  flex: 1;
  padding: var(--space-6);
  max-width: 1400px;
  width: 100%;
}

/* ── Page header ─────────────────────────────────── */
.page-header { margin-bottom: var(--space-6); }

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.025em;
  margin-bottom: 0.375rem;
}

.page-header p {
  color: var(--text-secondary);
  font-size: 0.938rem;
}

/* ── Stats grid ──────────────────────────────────── */
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
  transition: box-shadow 0.15s, border-color 0.15s;
}

.stat-card:hover {
  border-color: var(--border-default);
  box-shadow: var(--shadow-md);
}

.stat-label {
  font-size: 0.75rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: var(--text-muted);
  margin-bottom: var(--space-2);
}

.stat-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.025em;
}

.stat-card.success .stat-value { color: #059669; }
.stat-card.warning .stat-value { color: #d97706; }
.stat-card.danger  .stat-value { color: #dc2626; }
.stat-card.info    .stat-value { color: var(--color-accent); }

/* ── Cards ───────────────────────────────────────── */
.card {
  background: var(--surface-card);
  border-radius: var(--radius-lg);
  padding: var(--space-5);
  border: 1px solid var(--border-subtle);
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
  font-size: 1rem;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.01em;
}

/* ── Tables ──────────────────────────────────────── */
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
  font-size: 0.75rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: var(--text-muted);
}

td {
  padding: var(--space-3) var(--space-4);
  border-top: 1px solid #f1f5f9;
  color: var(--text-secondary);
  font-size: 0.875rem;
}

tbody tr { transition: background-color 0.1s; }
tbody tr:hover { background: var(--surface-card-hover); }

/* ── Badges ──────────────────────────────────────── */
.badge {
  display: inline-block;
  padding: 0.2rem 0.6rem;
  border-radius: 5px;
  font-size: 0.75rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.badge.success    { background: var(--status-success-bg);  color: var(--status-success-text); }
.badge.warning    { background: var(--status-warning-bg);  color: var(--status-warning-text); }
.badge.danger     { background: var(--status-danger-bg);   color: var(--status-danger-text);  }
.badge.info       { background: var(--status-info-bg);     color: var(--status-info-text);    }
.badge.increasing { background: var(--status-success-bg);  color: var(--status-success-text); }
.badge.decreasing { background: var(--status-danger-bg);   color: var(--status-danger-text);  }
.badge.stable     { background: #e0e7ff; color: #3730a3; }
.badge.high       { background: var(--status-danger-bg);   color: var(--status-danger-text);  }
.badge.medium     { background: var(--status-warning-bg);  color: var(--status-warning-text); }
.badge.low        { background: var(--status-info-bg);     color: var(--status-info-text);    }

/* ── State: loading / error ──────────────────────── */
.loading {
  text-align: center;
  padding: var(--space-10);
  color: var(--text-muted);
  font-size: 0.938rem;
}

.error {
  background: var(--status-danger-bg);
  border: 1px solid #fca5a5;
  color: var(--status-danger-text);
  padding: var(--space-4);
  border-radius: var(--radius-md);
  margin: var(--space-4) 0;
  font-size: 0.938rem;
}

/* ── Sidebar tooltip portal ──────────────────────── */
.sidebar-tooltip-portal {
  position: fixed;
  transform: translateY(-50%);
  background: #1e293b;
  color: #f1f5f9;
  font-size: 0.75rem;
  font-weight: 600;
  padding: 5px 10px;
  border-radius: 6px;
  white-space: nowrap;
  pointer-events: none;
  z-index: 9999;
  border: 1px solid #334155;
  box-shadow: 0 4px 6px -1px rgba(0,0,0,.3);
}

.sidebar-tooltip-portal::before {
  content: '';
  position: absolute;
  right: 100%;
  top: 50%;
  transform: translateY(-50%);
  border: 5px solid transparent;
  border-right-color: #334155;
}

/* ── Responsive ──────────────────────────────────── */
@media (max-width: 767px) {
  /* On mobile the sidebar is fixed/overlaid — body takes full width */
  .app-body {
    margin-left: 56px;  /* matches collapsed icon-strip width */
    width: calc(100% - 56px);
  }
}
</style>
