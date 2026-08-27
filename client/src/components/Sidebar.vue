<template>
  <aside
    class="sidebar"
    :class="{ collapsed, 'mobile-open': props.mobileOpen }"
  >
    <div class="sidebar-header">
      <div class="brand">
        <span class="brand-logo" v-html="greyhoundIcon"></span>
        <div class="brand-text" v-if="!collapsed">
          <h1>{{ t("nav.companyName") }}</h1>
          <span class="subtitle">{{ t("nav.subtitle") }}</span>
        </div>
      </div>
      <button
        v-if="!isTabletAuto"
        class="collapse-toggle"
        type="button"
        :aria-label="collapsed ? 'Expand sidebar' : 'Collapse sidebar'"
        @click="toggleCollapsed"
      >
        <svg width="18" height="18" viewBox="0 0 18 18" fill="none">
          <path
            v-if="!collapsed"
            d="M11.5 3.5L6 9L11.5 14.5"
            stroke="currentColor"
            stroke-width="1.75"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
          <path
            v-else
            d="M6.5 3.5L12 9L6.5 14.5"
            stroke="currentColor"
            stroke-width="1.75"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
        </svg>
      </button>
    </div>

    <nav class="nav-list">
      <router-link
        v-for="item in navItems"
        :key="item.to"
        :to="item.to"
        class="nav-item"
        :title="collapsed ? item.label : null"
        @click="$emit('close-mobile')"
      >
        <span class="nav-icon" v-html="item.icon"></span>
        <span v-if="!collapsed" class="nav-label">{{ item.label }}</span>
      </router-link>
    </nav>

    <div class="sidebar-footer">
      <LanguageSwitcher />
      <ProfileMenu
        @show-profile-details="$emit('show-profile-details')"
        @show-tasks="$emit('show-tasks')"
      />
    </div>
  </aside>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from "vue";
import { useI18n } from "../composables/useI18n";
import LanguageSwitcher from "./LanguageSwitcher.vue";
import ProfileMenu from "./ProfileMenu.vue";

const props = defineProps({
  mobileOpen: {
    type: Boolean,
    default: false,
  },
});

const emit = defineEmits([
  "show-profile-details",
  "show-tasks",
  "update:collapsed",
  "close-mobile",
]);

const { t } = useI18n();

// Manual collapse state, controlled by the user via the toggle button (desktop only, > 1024px)
const manualCollapsed = ref(false);
// True when the viewport is in the tablet auto-collapse range (769px - 1024px)
const isTabletAuto = ref(false);
// True when the viewport is in the off-canvas mobile drawer range (<= 768px)
const isMobileRange = ref(false);

let tabletMql = null;
let mobileMql = null;
const updateTabletMode = (e) => {
  isTabletAuto.value = e.matches;
};
const updateMobileMode = (e) => {
  isMobileRange.value = e.matches;
};

onMounted(() => {
  tabletMql = window.matchMedia("(min-width: 769px) and (max-width: 1024px)");
  isTabletAuto.value = tabletMql.matches;
  tabletMql.addEventListener("change", updateTabletMode);

  mobileMql = window.matchMedia("(max-width: 768px)");
  isMobileRange.value = mobileMql.matches;
  mobileMql.addEventListener("change", updateMobileMode);
});

onUnmounted(() => {
  if (tabletMql) {
    tabletMql.removeEventListener("change", updateTabletMode);
  }
  if (mobileMql) {
    mobileMql.removeEventListener("change", updateMobileMode);
  }
});

// Below 768px the off-canvas drawer always opens fully expanded.
// Between 769-1024px the sidebar is always icon-only regardless of manual state.
// Above 1024px the manual toggle controls the state.
const collapsed = computed(() => {
  if (isMobileRange.value) return false;
  if (isTabletAuto.value) return true;
  return manualCollapsed.value;
});

const toggleCollapsed = () => {
  manualCollapsed.value = !manualCollapsed.value;
  emit("update:collapsed", manualCollapsed.value);
};

const greyhoundIcon = `<svg width="24" height="24" viewBox="0 0 32 32" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M25.5 8.5C25.5 9.6 24.6 10.5 23.5 10.5C22.9 10.5 22.4 10.25 22.05 9.85L20 11V13.5L23.5 15.5C24.05 15.8 24.4 16.4 24.35 17.05L23.9 22.5C23.85 23.05 23.4 23.5 22.85 23.5C22.3 23.5 21.85 23.05 21.85 22.5L21.7 18L19.5 16.7V19.5L21 24.5C21.2 25.15 20.75 25.8 20.05 25.8C19.55 25.8 19.1 25.45 18.95 24.95L17.3 20H14.2L13 24.9C12.87 25.42 12.4 25.8 11.85 25.8C11.15 25.8 10.65 25.1 10.9 24.45L12.5 20V16L10.5 17.5L9.9 21.5C9.82 22.05 9.35 22.45 8.8 22.4C8.2 22.35 7.77 21.8 7.85 21.2L8.5 16.5C8.57 16 8.85 15.55 9.27 15.27L12.5 13V10.5L10.7 9C10.35 9.3 9.9 9.5 9.4 9.5C8.3 9.5 7.4 8.6 7.4 7.5C7.4 6.4 8.3 5.5 9.4 5.5C10.4 5.5 11.22 6.24 11.37 7.2L14.5 9.5H17.5L20.6 7.25C20.73 6.26 21.57 5.5 22.6 5.5C23.7 5.5 24.6 6.4 24.6 7.5C24.6 7.63 24.59 7.75 24.57 7.87L25.5 8.5Z" fill="currentColor"/></svg>`;

const icons = {
  overview: `<svg width="20" height="20" viewBox="0 0 20 20" fill="none"><path d="M3 8.5L10 3L17 8.5" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/><path d="M5 7.5V16C5 16.5523 5.44772 17 6 17H14C14.5523 17 15 16.5523 15 16V7.5" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/><path d="M8 17V12H12V17" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>`,
  inventory: `<svg width="20" height="20" viewBox="0 0 20 20" fill="none"><path d="M3 6L10 3L17 6L10 9L3 6Z" stroke="currentColor" stroke-width="1.5" stroke-linejoin="round"/><path d="M3 6V14L10 17L17 14V6" stroke="currentColor" stroke-width="1.5" stroke-linejoin="round"/><path d="M10 9V17" stroke="currentColor" stroke-width="1.5"/></svg>`,
  orders: `<svg width="20" height="20" viewBox="0 0 20 20" fill="none"><path d="M6 4H14C14.5523 4 15 4.44772 15 5V16C15 16.5523 14.5523 17 14 17H6C5.44772 17 5 16.5523 5 16V5C5 4.44772 5.44772 4 6 4Z" stroke="currentColor" stroke-width="1.5"/><path d="M8 3H12V5H8V3Z" stroke="currentColor" stroke-width="1.5" stroke-linejoin="round"/><path d="M7.5 9H12.5" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/><path d="M7.5 12H12.5" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/></svg>`,
  finance: `<svg width="20" height="20" viewBox="0 0 20 20" fill="none"><path d="M10 3V17" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/><path d="M13.5 6C13.5 6 12.5 5 10 5C7.5 5 6.5 6.2 6.5 7.3C6.5 10 13.5 9 13.5 11.7C13.5 12.8 12.5 14 10 14C7.5 14 6.5 13 6.5 13" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/></svg>`,
  demand: `<svg width="20" height="20" viewBox="0 0 20 20" fill="none"><path d="M3 14L8 9L11.5 12.5L17 6" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/><path d="M12.5 6H17V10.5" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>`,
  reports: `<svg width="20" height="20" viewBox="0 0 20 20" fill="none"><path d="M5 3H12L15 6V17H5V3Z" stroke="currentColor" stroke-width="1.5" stroke-linejoin="round"/><path d="M12 3V6H15" stroke="currentColor" stroke-width="1.5" stroke-linejoin="round"/><path d="M7.5 10H12.5" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/><path d="M7.5 13H12.5" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/></svg>`,
  backlog: `<svg width="20" height="20" viewBox="0 0 20 20" fill="none"><path d="M6 4H14C14.5523 4 15 4.44772 15 5V16C15 16.5523 14.5523 17 14 17H6C5.44772 17 5 16.5523 5 16V5C5 4.44772 5.44772 4 6 4Z" stroke="currentColor" stroke-width="1.5"/><path d="M8 3H12V5H8V3Z" stroke="currentColor" stroke-width="1.5" stroke-linejoin="round"/><path d="M7.5 10.5L9 12L12.5 8" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>`,
};

const navItems = computed(() => [
  { to: "/", label: t("nav.overview"), icon: icons.overview },
  { to: "/inventory", label: t("nav.inventory"), icon: icons.inventory },
  { to: "/orders", label: t("nav.orders"), icon: icons.orders },
  { to: "/spending", label: t("nav.finance"), icon: icons.finance },
  { to: "/demand", label: t("nav.demandForecast"), icon: icons.demand },
  { to: "/reports", label: t("nav.reports"), icon: icons.reports },
  { to: "/backlog", label: t("nav.backlog"), icon: icons.backlog },
]);
</script>

<style scoped>
.sidebar {
  grid-row: 1 / -1;
  display: flex;
  flex-direction: column;
  width: var(--sidebar-width);
  height: 100vh;
  position: sticky;
  top: 0;
  background: var(--color-surface);
  border-right: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
  transition: width 0.2s ease;
  overflow: hidden;
  z-index: 100;
}

.sidebar.collapsed {
  width: var(--sidebar-width-collapsed);
}

.sidebar-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--space-2);
  padding: var(--space-5) var(--space-4);
  border-bottom: 1px solid var(--color-border);
  min-height: 70px;
}

.sidebar.collapsed .sidebar-header {
  justify-content: center;
}

.brand {
  display: flex;
  align-items: center;
  gap: var(--space-2);
  min-width: 0;
}

.brand-logo {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  width: 28px;
  height: 28px;
  color: var(--color-accent);
}

.brand-logo :deep(svg) {
  width: 100%;
  height: 100%;
}

.brand-text {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  min-width: 0;
}

.brand h1 {
  font-size: 1.125rem;
  font-weight: 700;
  color: var(--color-text-primary);
  letter-spacing: -0.025em;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.brand .subtitle {
  font-size: 0.75rem;
  color: var(--color-text-secondary);
  font-weight: 400;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.collapse-toggle {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: var(--radius-sm);
  border: 1px solid var(--color-border);
  background: var(--color-surface);
  color: var(--color-text-secondary);
  cursor: pointer;
  transition: all 0.15s ease;
}

.collapse-toggle:hover {
  background: var(--color-bg);
  color: var(--color-text-primary);
}

.nav-list {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  padding: var(--space-4) var(--space-3);
  overflow-y: auto;
  overflow-x: hidden;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-3) var(--space-3);
  border-radius: var(--radius-sm);
  color: var(--color-text-secondary);
  text-decoration: none;
  font-weight: 500;
  font-size: 0.9rem;
  white-space: nowrap;
  transition:
    background-color 0.15s ease,
    color 0.15s ease;
}

.sidebar.collapsed .nav-item {
  justify-content: center;
}

.nav-item:hover {
  background: var(--color-bg);
  color: var(--color-text-primary);
}

.nav-item.router-link-active {
  background: var(--color-accent-soft);
  color: var(--color-accent);
}

.nav-item.router-link-active:hover {
  color: var(--color-accent-strong);
}

.nav-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.nav-label {
  overflow: hidden;
  text-overflow: ellipsis;
}

.sidebar-footer {
  margin-top: auto;
  display: flex;
  flex-direction: column;
  gap: var(--space-3);
  padding: var(--space-4) var(--space-3);
  border-top: 1px solid var(--color-border);
}

.sidebar.collapsed .sidebar-footer {
  align-items: center;
}

.sidebar.collapsed .sidebar-footer :deep(.language-label),
.sidebar.collapsed .sidebar-footer :deep(.profile-name) {
  display: none;
}

@media (max-width: 768px) {
  .sidebar,
  .sidebar.collapsed {
    position: fixed;
    top: 0;
    left: 0;
    width: var(--sidebar-width);
    height: 100vh;
    transform: translateX(-100%);
    z-index: 200;
    transition: transform 0.2s ease;
  }

  .sidebar.mobile-open,
  .sidebar.collapsed.mobile-open {
    transform: translateX(0);
  }

  .sidebar.collapsed .sidebar-header {
    justify-content: space-between;
  }

  .sidebar.collapsed .nav-item {
    justify-content: flex-start;
  }

  .sidebar.collapsed .sidebar-footer {
    align-items: stretch;
  }

  .sidebar.collapsed .sidebar-footer :deep(.language-label),
  .sidebar.collapsed .sidebar-footer :deep(.profile-name) {
    display: inline;
  }
}
</style>
