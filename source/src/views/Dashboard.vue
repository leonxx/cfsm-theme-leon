<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue'
import {
  initMonitor,
  monitor,
  refreshDashboard,
  registerVisibilityHandler,
  retryMonitor,
  setView,
  stopAllSockets
} from '../stores/monitor'
import { t } from '../utils/i18n'
import AppHeader from '../components/AppHeader.vue'
import AppIcon from '../components/AppIcon.vue'
import SummaryHero from '../components/SummaryHero.vue'
import ServerCard from '../components/ServerCard.vue'
import ServerTable from '../components/ServerTable.vue'
import FooterBar from '../components/FooterBar.vue'

const retrying = ref(false)
let unregisterVisibility = null

async function retry() {
  retrying.value = true
  await retryMonitor()
  retrying.value = false
}

onMounted(() => {
  initMonitor()
  unregisterVisibility = registerVisibilityHandler(async () => {
    await refreshDashboard()
  })
})

onBeforeUnmount(() => {
  unregisterVisibility?.()
  stopAllSockets()
})
</script>

<template>
  <div class="dashboard-page">
    <AppHeader />

    <main class="container dashboard-main">
      <template v-if="monitor.loading && !monitor.ready">
        <div class="summary-grid">
          <div v-for="n in 4" :key="n" class="skeleton tile-skeleton"></div>
        </div>
        <div class="server-grid">
          <div v-for="n in 6" :key="n" class="skeleton card-skeleton"></div>
        </div>
      </template>

      <template v-else-if="monitor.error && !monitor.ready">
        <div class="error-panel">
          <span class="panel-mark"><AppIcon name="alert" :size="24" /></span>
          <h2>{{ t('loadFailed') }}</h2>
          <p>{{ monitor.error || t('loadFailedHint') }}</p>
          <button class="btn btn-primary" type="button" :disabled="retrying" @click="retry">
            <AppIcon name="refresh" :size="15" :class="{ spin: retrying }" />
            {{ t('retry') }}
          </button>
        </div>
      </template>

      <template v-else>
        <SummaryHero />

        <div v-if="monitor.servers.length" class="server-block">
          <div class="server-toolbar">
            <span class="toolbar-label">
              <AppIcon name="server" :size="14" />
              {{ t('nodes') }} · {{ monitor.servers.length }}
            </span>
            <div class="view-switch" role="group" :aria-label="t('nodes')">
              <button
                type="button"
                :class="{ active: monitor.view === 'grid' }"
                :aria-pressed="monitor.view === 'grid'"
                @click="setView('grid')"
              >
                <AppIcon name="grid" :size="14" />
                <span>{{ t('gridView') }}</span>
              </button>
              <button
                type="button"
                :class="{ active: monitor.view === 'list' }"
                :aria-pressed="monitor.view === 'list'"
                @click="setView('list')"
              >
                <AppIcon name="list" :size="14" />
                <span>{{ t('listView') }}</span>
              </button>
            </div>
          </div>

          <ServerTable v-if="monitor.view === 'list'" :servers="monitor.servers" />

          <div v-else class="server-grid">
            <ServerCard
              v-for="(server, index) in monitor.servers"
              :key="server.id"
              :server="server"
              :index="index"
            />
          </div>
        </div>

        <div v-else class="empty-panel">
          <span class="panel-mark"><AppIcon name="server" :size="24" /></span>
          <h2>{{ t('noNodes') }}</h2>
          <p>{{ t('noNodesHint') }}</p>
          <button class="btn" type="button" :disabled="retrying" @click="retry">
            <AppIcon name="refresh" :size="14" />
            {{ t('retry') }}
          </button>
        </div>

        <FooterBar />
      </template>
    </main>
  </div>
</template>

<style scoped>
.dashboard-main {
  padding-top: 16px;
}

.summary-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: var(--gap);
  margin-bottom: var(--gap);
}

.server-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  align-items: start;
  gap: var(--gap);
}

.server-block {
  display: flex;
  flex-direction: column;
  gap: var(--gap);
}

.server-toolbar {
  display: flex;
  min-height: 30px;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.toolbar-label {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  color: var(--muted);
  font-size: 11.5px;
  font-weight: 600;
  letter-spacing: 0.04em;
  text-transform: uppercase;
}

.view-switch {
  display: inline-flex;
  gap: 2px;
  padding: 3px;
  border: 1px solid var(--line);
  border-radius: 9px;
  background: var(--bg-soft);
}

.view-switch button {
  display: inline-flex;
  height: 26px;
  align-items: center;
  gap: 5px;
  padding: 0 9px;
  border-radius: 6px;
  color: var(--muted);
  font-size: 11.5px;
  font-weight: 550;
  transition: background 0.18s ease, color 0.18s ease;
}

.view-switch button:hover {
  color: var(--text-soft);
}

.view-switch button.active {
  background: var(--surface);
  color: var(--text);
  box-shadow: var(--shadow-soft);
}

.tile-skeleton {
  min-height: 110px;
  border-radius: var(--radius);
}

.card-skeleton {
  height: 286px;
  border-radius: var(--radius);
}

.error-panel,
.empty-panel {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 64px 24px;
  border: 1px solid var(--line);
  border-radius: var(--radius);
  background: var(--surface);
  text-align: center;
}

.panel-mark {
  display: grid;
  width: 54px;
  height: 54px;
  margin-bottom: 16px;
  place-items: center;
  border: 1px solid var(--line);
  border-radius: 14px;
  background: var(--bg-soft);
  color: var(--muted);
}

.error-panel .panel-mark {
  border-color: rgba(var(--bad-rgb), 0.24);
  background: rgba(var(--bad-rgb), 0.07);
  color: var(--bad);
}

.error-panel h2,
.empty-panel h2 {
  margin: 0 0 6px;
  font-size: 18px;
  font-weight: 600;
  letter-spacing: -0.02em;
}

.error-panel p,
.empty-panel p {
  max-width: 360px;
  margin: 0 0 18px;
  color: var(--muted);
  font-size: 12px;
  line-height: 1.6;
}

@media (max-width: 1279px) {
  .server-grid {
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }
}

@media (max-width: 1023px) {
  .server-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .summary-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

@media (max-width: 639px) {
  .server-grid {
    grid-template-columns: 1fr;
  }

  .view-switch button span {
    display: none;
  }

  .view-switch button {
    padding: 0 7px;
  }
}
</style>
