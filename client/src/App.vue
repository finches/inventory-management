<template>
  <div class="app" :class="{ 'sidebar-collapsed': sidebarCollapsed }">
    <div class="mobile-topbar">
      <button
        class="hamburger-btn"
        type="button"
        aria-label="Open menu"
        @click="mobileSidebarOpen = true"
      >
        <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
          <path d="M3 5H17" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" />
          <path d="M3 10H17" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" />
          <path d="M3 15H17" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" />
        </svg>
      </button>
      <span class="mobile-brand-logo" v-html="greyhoundIcon"></span>
      <span class="mobile-brand">{{ t('nav.companyName') }}</span>
    </div>

    <div
      class="sidebar-backdrop"
      :class="{ open: mobileSidebarOpen }"
      @click="mobileSidebarOpen = false"
    ></div>

    <Sidebar
      :mobile-open="mobileSidebarOpen"
      @update:collapsed="sidebarCollapsed = $event"
      @show-profile-details="showProfileDetails = true"
      @show-tasks="showTasks = true"
      @close-mobile="mobileSidebarOpen = false"
    />

    <div class="content-area">
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
import { ref, computed } from 'vue'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import Sidebar from './components/Sidebar.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileDetailsModal,
    TasksModal,
    Sidebar
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const sidebarCollapsed = ref(false)
    const mobileSidebarOpen = ref(false)
    const greyhoundIcon = `<svg width="24" height="24" viewBox="0 0 32 32" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M25.5 8.5C25.5 9.6 24.6 10.5 23.5 10.5C22.9 10.5 22.4 10.25 22.05 9.85L20 11V13.5L23.5 15.5C24.05 15.8 24.4 16.4 24.35 17.05L23.9 22.5C23.85 23.05 23.4 23.5 22.85 23.5C22.3 23.5 21.85 23.05 21.85 22.5L21.7 18L19.5 16.7V19.5L21 24.5C21.2 25.15 20.75 25.8 20.05 25.8C19.55 25.8 19.1 25.45 18.95 24.95L17.3 20H14.2L13 24.9C12.87 25.42 12.4 25.8 11.85 25.8C11.15 25.8 10.65 25.1 10.9 24.45L12.5 20V16L10.5 17.5L9.9 21.5C9.82 22.05 9.35 22.45 8.8 22.4C8.2 22.35 7.77 21.8 7.85 21.2L8.5 16.5C8.57 16 8.85 15.55 9.27 15.27L12.5 13V10.5L10.7 9C10.35 9.3 9.9 9.5 9.4 9.5C8.3 9.5 7.4 8.6 7.4 7.5C7.4 6.4 8.3 5.5 9.4 5.5C10.4 5.5 11.22 6.24 11.37 7.2L14.5 9.5H17.5L20.6 7.25C20.73 6.26 21.57 5.5 22.6 5.5C23.7 5.5 24.6 6.4 24.6 7.5C24.6 7.63 24.59 7.75 24.57 7.87L25.5 8.5Z" fill="currentColor"/></svg>`

    const tasks = computed(() => currentUser.value.tasks)

    const addTask = (taskData) => {
      // No backend task storage exists; tasks live only on the mock user
      currentUser.value.tasks.unshift({
        id: `task-${Date.now()}`,
        status: 'pending',
        ...taskData
      })
    }

    const deleteTask = (taskId) => {
      const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
      if (index !== -1) {
        currentUser.value.tasks.splice(index, 1)
      }
    }

    const toggleTask = (taskId) => {
      const task = currentUser.value.tasks.find(t => t.id === taskId)
      if (task) {
        task.status = task.status === 'pending' ? 'completed' : 'pending'
      }
    }

    return {
      t,
      greyhoundIcon,
      showProfileDetails,
      showTasks,
      sidebarCollapsed,
      mobileSidebarOpen,
      tasks,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
:root {
  --space-1: 0.25rem; --space-2: 0.5rem; --space-3: 0.75rem; --space-4: 1rem;
  --space-5: 1.25rem; --space-6: 1.5rem; --space-8: 2rem;
  --radius-sm: 6px; --radius-md: 10px; --radius-lg: 14px;
  --shadow-sm: 0 1px 3px 0 rgba(0,0,0,0.05);
  --shadow-md: 0 4px 12px rgba(0,0,0,0.06);
  --shadow-lg: 0 10px 25px rgba(0,0,0,0.08);
  --color-bg: #FBF6EF; --color-surface: #FFFFFF; --color-border: #E7D9C4;
  --color-text-primary: #2B2118; --color-text-secondary: #7C6E5C;
  --color-accent: #A9702F; --color-accent-strong: #82551F; --color-accent-soft: #F1E1C9;
  --sidebar-width: 260px; --sidebar-width-collapsed: 72px;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  background: var(--color-bg);
  color: var(--color-text-primary);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.app {
  display: grid;
  grid-template-columns: var(--sidebar-width) 1fr;
  min-height: 100vh;
  transition: grid-template-columns 0.2s ease;
}

.app.sidebar-collapsed {
  grid-template-columns: var(--sidebar-width-collapsed) 1fr;
}

.content-area {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.main-content {
  flex: 1;
  max-width: 1600px;
  width: 100%;
  margin: 0 auto;
  padding: 1.5rem 2rem;
}

.mobile-topbar {
  display: none;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-3) var(--space-4);
  background: var(--color-surface);
  border-bottom: 1px solid var(--color-border);
  position: sticky;
  top: 0;
  z-index: 150;
}

.hamburger-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  border-radius: var(--radius-sm);
  border: 1px solid var(--color-border);
  background: var(--color-surface);
  color: var(--color-text-secondary);
  cursor: pointer;
  transition: all 0.15s ease;
}

.hamburger-btn:hover {
  background: var(--color-bg);
  color: var(--color-text-primary);
}

.mobile-brand-logo {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  color: var(--color-accent);
  flex-shrink: 0;
}

.mobile-brand-logo svg {
  width: 100%;
  height: 100%;
}

.mobile-brand {
  font-size: 1.05rem;
  font-weight: 700;
  color: var(--color-text-primary);
  letter-spacing: -0.025em;
}

.sidebar-backdrop {
  display: none;
}

.page-header {
  margin-bottom: var(--space-6);
}

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: var(--color-text-primary);
  margin-bottom: var(--space-2);
  letter-spacing: -0.025em;
}

.page-header p {
  color: var(--color-text-secondary);
  font-size: 0.938rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  background: var(--color-surface);
  padding: var(--space-5);
  border-radius: var(--radius-md);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
  transition: box-shadow 0.15s ease, transform 0.15s ease, border-color 0.15s ease;
}

.stat-card:hover {
  border-color: var(--color-accent);
  box-shadow: var(--shadow-md);
  transform: translateY(-1px);
}

.stat-label {
  color: var(--color-text-secondary);
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 0.625rem;
}

.stat-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: var(--color-text-primary);
  letter-spacing: -0.025em;
}

.stat-card.warning .stat-value {
  color: #ea580c;
}

.stat-card.success .stat-value {
  color: #059669;
}

.stat-card.danger .stat-value {
  color: #dc2626;
}

.stat-card.info .stat-value {
  color: var(--color-accent);
}

.card {
  background: var(--color-surface);
  border-radius: var(--radius-md);
  padding: var(--space-5);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
  margin-bottom: var(--space-5);
  transition: box-shadow 0.15s ease, transform 0.15s ease;
}

.card:hover {
  box-shadow: var(--shadow-md);
  transform: translateY(-1px);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--space-4);
  padding-bottom: var(--space-3);
  border-bottom: 1px solid var(--color-border);
}

.card-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: var(--color-text-primary);
  letter-spacing: -0.025em;
}

.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: var(--color-bg);
  border-top: 1px solid var(--color-border);
  border-bottom: 1px solid var(--color-border);
}

th {
  text-align: left;
  padding: var(--space-2) var(--space-3);
  font-weight: 600;
  color: var(--color-text-secondary);
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

td {
  padding: var(--space-2) var(--space-3);
  border-top: 1px solid var(--color-border);
  color: var(--color-text-primary);
  font-size: 0.875rem;
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: var(--color-bg);
}

.badge {
  display: inline-block;
  padding: var(--space-1) var(--space-3);
  border-radius: var(--radius-sm);
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success {
  background: #d1fae5;
  color: #065f46;
}

.badge.warning {
  background: #fed7aa;
  color: #92400e;
}

.badge.danger {
  background: #fecaca;
  color: #991b1b;
}

.badge.info {
  background: #dbeafe;
  color: #1e40af;
}

.badge.increasing {
  background: #d1fae5;
  color: #065f46;
}

.badge.decreasing {
  background: #fecaca;
  color: #991b1b;
}

.badge.stable {
  background: #e0e7ff;
  color: #3730a3;
}

.badge.high {
  background: #fecaca;
  color: #991b1b;
}

.badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.badge.low {
  background: #dbeafe;
  color: #1e40af;
}

.loading {
  text-align: center;
  padding: 3rem;
  color: var(--color-text-secondary);
  font-size: 0.938rem;
}

.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}

@media (max-width: 768px) {
  .app,
  .app.sidebar-collapsed {
    grid-template-columns: 1fr;
  }

  .mobile-topbar {
    display: flex;
  }

  .sidebar-backdrop.open {
    display: block;
    position: fixed;
    inset: 0;
    background: rgba(43, 33, 24, 0.4);
    z-index: 150;
  }

  .main-content {
    padding: var(--space-4);
  }
}
</style>
