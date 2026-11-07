<script setup>
import { computed } from 'vue'
import { RouterLink } from 'vue-router'
import { monitor } from '../stores/monitor'
import { locale, t } from '../utils/i18n'
import {
  expireInfo,
  formatBytes,
  formatNumber,
  formatSpeed,
  isServerOnline,
  latencyRoutes,
  monthTraffic,
  pairBytes,
  regionCode,
  serverDiskPercent,
  serverRamPercent,
  serverTimestamp,
  serverUptimeSeconds,
  trafficLimitBytes,
  trafficPercent
} from '../utils/format'
import AppIcon from './AppIcon.vue'

const props = defineProps({
  server: { type: Object, required: true },
  index: { type: Number, default: 0 }
})

const sys = computed(() => monitor.sysConfig)
const online = computed(() => isServerOnline(props.server, monitor.now))
const region = computed(() => regionCode(props.server.region))

const osText = computed(() => {
  const os = String(props.server.os || '').trim()
  const arch = String(props.server.arch || '').trim()
  return [os, arch].filter(Boolean).join(' · ') || '—'
})

const expire = computed(() => (sys.value.show_expire
  ? expireInfo(props.server.expire_date, locale.value, monitor.now)
  : null))

const uptime = computed(() => {
  if (!online.value) {
    const seen = serverTimestamp(props.server)
    if (!seen) return t('notConnected')
    const seconds = Math.max(0, Math.round((monitor.now - seen) / 1000))
    return `${t('offline')} ${compactDuration(seconds)}`
  }
  const seconds = serverUptimeSeconds(props.server, monitor.now)
  return seconds > 0 ? `${t('online')} ${compactDuration(seconds)}` : t('online')
})

function compactDuration(seconds) {
  const total = Math.max(0, Math.floor(Number(seconds) || 0))
  const days = Math.floor(total / 86400)
  const hours = Math.floor((total % 86400) / 3600)
  const minutes = Math.floor((total % 3600) / 60)
  if (locale.value === 'en') {
    if (days > 0) return `${days}d`
    if (hours > 0) return `${hours}h`
    return `${minutes}m`
  }
  if (days > 0) return `${days} 天`
  if (hours > 0) return `${hours} 小时`
  return `${minutes} 分`
}

const fault = computed(() => {
  if (!online.value) return true
  const losses = ['loss_ct', 'loss_cu', 'loss_cm']
    .map((key) => Number(props.server[key]))
    .filter((value) => Number.isFinite(value))
  if (losses.some((value) => value >= 50)) return true
  const configured = latencyRoutes(props.server)
    .filter((route) => route.key !== 'bd' && route.state !== 'off')
  return configured.length > 0 && configured.every((route) => route.state === 'timeout')
})

const cpuCores = computed(() => Number(props.server.cpu_cores) || 0)
const cpuLabel = computed(() => (cpuCores.value ? `CPU ${cpuCores.value} 核` : 'CPU'))
const cpuPct = computed(() => {
  const value = Number(props.server.cpu)
  return online.value && Number.isFinite(value) ? Math.max(0, Math.min(100, value)) : null
})

const loadText = computed(() => {
  const raw = String(props.server.load_avg || '').trim()
  if (raw) return raw
  const values = [props.server.load_1, props.server.load_5, props.server.load_15]
    .map((value) => Number(value))
    .filter((value) => Number.isFinite(value))
  return values.length ? values.map((value) => value.toFixed(2)).join(' ') : '—'
})

const ramPct = computed(() => (online.value ? serverRamPercent(props.server) : null))
const diskPct = computed(() => (online.value ? serverDiskPercent(props.server) : null))

const ramFoot = computed(() => {
  const used = (Number(props.server.ram_used) || 0) * 1024 * 1024
  const total = (Number(props.server.ram_total) || 0) * 1024 * 1024
  if (online.value && total > 0) return pairBytes(used, total)
  return total > 0 ? formatBytes(total) : '—'
})

const diskFoot = computed(() => {
  const used = (Number(props.server.disk_used) || 0) * 1024 * 1024
  const total = (Number(props.server.disk_total) || 0) * 1024 * 1024
  if (online.value && total > 0) return pairBytes(used, total)
  return total > 0 ? formatBytes(total) : '—'
})

const limit = computed(() => trafficLimitBytes(props.server))
const usedTraffic = computed(() => monthTraffic(props.server))
const trafficPct = computed(() => (limit.value > 0 ? trafficPercent(props.server) : null))
const trafficFoot = computed(() => {
  if (limit.value > 0) return pairBytes(usedTraffic.value, limit.value)
  return `${formatBytes(usedTraffic.value)} / ∞`
})

const pings = computed(() => {
  const routes = latencyRoutes(props.server)
  const names = {
    ct: sys.value.custom_ct_name || t('telecom'),
    cu: sys.value.custom_cu_name || t('unicom'),
    cm: sys.value.custom_cm_name || t('mobile')
  }
  return ['ct', 'cm', 'cu'].map((key) => {
    const route = routes.find((item) => item.key === key)
    const state = route?.state || 'off'
    const value = Number(route?.value)
    const loss = props.server[route.loss]
    const lossText = loss === null
      ? t('timeout')
      : loss !== false && loss !== undefined && loss !== '' && Number.isFinite(Number(loss))
        ? `${Number(loss).toFixed(1)}%`
        : ''
    const latencyText = state === 'ok' && Number.isFinite(value)
      ? `${value}ms`
      : state === 'timeout' ? t('timeout') : '—'
    return {
      key,
      name: names[key],
      state,
      text: `${latencyText}${lossText ? `（${lossText}）` : ''}`
    }
  }).filter((item) => item.state !== 'off')
})

const showPings = computed(() => Boolean(sys.value.show_three_net_details)
  && pings.value.some((item) => item.state !== 'off'))
const showTraffic = computed(() => Boolean(sys.value.show_tf))

function pctText(value, fallback = '—') {
  if (value === null) return fallback
  const number = Math.max(0, Math.min(100, Number(value) || 0))
  return `${number < 10 ? number.toFixed(1) : number.toFixed(0)}%`
}
</script>

<template>
  <RouterLink
    :to="`/server/${encodeURIComponent(server.id)}`"
    class="node-card"
    :class="{ 'is-offline': !online }"
  >
    <div class="card-top">
      <div class="card-identity">
        <div class="name-row">
          <h3 :title="server.name">{{ server.name || server.id }}</h3>
          <span v-if="region" class="region-pill">{{ region }}</span>
          <span v-if="fault" class="fault-badge">{{ t('nodeFault') }}</span>
        </div>
        <p class="os-line" :title="osText">{{ osText }}</p>
      </div>

      <div class="card-status">
        <span class="state-pill" :class="online ? 'is-online' : 'is-offline'">
          <i></i>
          {{ uptime }}
        </span>
        <span v-if="expire" class="expire" :class="`tone-${expire.tone}`">{{ expire.text }}</span>
        <span v-else-if="sys.show_expire" class="expire">∞</span>
      </div>
    </div>

    <div class="meter-grid">
      <div class="meter">
        <div class="meter-head">
          <span class="meter-label">{{ cpuLabel }}</span>
          <span class="meter-pct">{{ pctText(cpuPct) }}</span>
        </div>
        <div class="meter-track"><i :style="{ width: `${cpuPct ?? 0}%` }"></i></div>
        <div class="meter-foot">{{ loadText }}</div>
      </div>

      <div class="meter">
        <div class="meter-head">
          <span class="meter-label">{{ t('memory') }}</span>
          <span class="meter-pct">{{ pctText(ramPct) }}</span>
        </div>
        <div class="meter-track"><i :style="{ width: `${ramPct ?? 0}%` }"></i></div>
        <div class="meter-foot">{{ ramFoot }}</div>
      </div>

      <div class="meter">
        <div class="meter-head">
          <span class="meter-label">{{ t('hardDisk') }}</span>
          <span class="meter-pct">{{ pctText(diskPct) }}</span>
        </div>
        <div class="meter-track"><i :style="{ width: `${diskPct ?? 0}%` }"></i></div>
        <div class="meter-foot">{{ diskFoot }}</div>
      </div>

      <div v-if="showTraffic" class="meter">
        <div class="meter-head">
          <span class="meter-label">{{ t('traffic') }}</span>
          <span class="meter-pct">{{ pctText(trafficPct, '∞') }}</span>
        </div>
        <div class="meter-track"><i :style="{ width: `${trafficPct ?? 0}%` }"></i></div>
        <div class="meter-foot">{{ trafficFoot }}</div>
      </div>
    </div>

    <div class="net-row">
      <span class="net-item">
        <AppIcon name="arrowDown" :size="12" />
        <span class="tnum">{{ formatSpeed(server.net_in_speed) }}</span>
      </span>
      <span class="net-item">
        <AppIcon name="arrowUp" :size="12" />
        <span class="tnum">{{ formatSpeed(server.net_out_speed) }}</span>
      </span>
      <span class="net-item net-total">
        <AppIcon name="arrowDown" :size="12" />
        <span class="tnum">{{ formatBytes(server.net_rx) }}</span>
      </span>
      <span class="net-item net-total">
        <AppIcon name="arrowUp" :size="12" />
        <span class="tnum">{{ formatBytes(server.net_tx) }}</span>
      </span>
      <span class="net-item tnum" :title="t('tcp')">T:{{ formatNumber(server.tcp_conn ?? 0) }}</span>
      <span class="net-item tnum" :title="t('udp')">U:{{ formatNumber(server.udp_conn ?? 0) }}</span>
    </div>

    <div v-if="showPings" class="ping-row">
      <AppIcon name="bolt" :size="12" class="ping-icon" />
      <div class="ping-list">
        <span v-for="item in pings" :key="item.key" class="ping-item tnum">
          {{ item.name }} <b :class="{ 'is-empty': item.state !== 'ok' }">{{ item.text }}</b>
        </span>
      </div>
    </div>
  </RouterLink>
</template>

<style scoped>
.node-card {
  container-type: inline-size;
  display: flex;
  min-width: 0;
  flex-direction: column;
  padding: 16px;
  border: 1px solid var(--line);
  border-radius: var(--radius);
  background: var(--surface);
  color: var(--text);
  box-shadow: var(--shadow-soft);
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
}

.node-card:hover {
  border-color: var(--line-strong);
  box-shadow: 0 4px 14px rgba(0, 0, 0, 0.08);
}

.node-card.is-offline {
  background: var(--bg-soft);
}

.card-top {
  display: flex;
  min-width: 0;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
}

.card-identity {
  min-width: 0;
}

.name-row {
  display: flex;
  min-width: 0;
  align-items: center;
  gap: 6px;
}

.name-row h3 {
  overflow: hidden;
  margin: 0;
  color: var(--text);
  font-size: 16px;
  font-weight: 500;
  line-height: 1.4;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.region-pill {
  display: inline-flex;
  flex: 0 0 auto;
  align-items: center;
  justify-content: center;
  padding: 2px 7px;
  border: 1px solid var(--line);
  border-radius: 999px;
  color: var(--muted);
  font-size: 11px;
  line-height: 1.3;
  white-space: nowrap;
}

.fault-badge {
  flex: 0 0 auto;
  padding: 1px 6px;
  border: 1px solid var(--bad);
  border-radius: 4px;
  background: var(--bad);
  color: #ffffff;
  font-size: 10px;
  font-weight: 700;
  line-height: 1.3;
  white-space: nowrap;
}

.os-line {
  overflow: hidden;
  margin: 4px 0 0;
  color: var(--muted);
  font-size: 12px;
  line-height: 1.4;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.card-status {
  display: flex;
  flex: 0 0 auto;
  flex-direction: column;
  align-items: flex-end;
  gap: 4px;
}

.state-pill {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 2px 7px;
  border: 1px solid var(--line);
  border-radius: 999px;
  color: var(--muted);
  font-size: 11px;
  line-height: 1.3;
  white-space: nowrap;
}

.state-pill i {
  width: 6px;
  height: 6px;
  flex: 0 0 auto;
  border-radius: 50%;
  background: currentColor;
}

.state-pill.is-online {
  border-color: rgba(var(--good-rgb), 0.3);
  background: rgba(var(--good-rgb), 0.08);
  color: var(--good);
}

.state-pill.is-offline {
  border-color: rgba(var(--bad-rgb), 0.3);
  background: rgba(var(--bad-rgb), 0.08);
  color: var(--bad);
}

.expire {
  color: var(--muted);
  font-size: 12px;
  font-variant-numeric: tabular-nums;
  white-space: nowrap;
}

.meter-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 16px;
  margin-top: 16px;
}

.meter {
  min-width: 0;
}

.meter-head {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: 8px;
}

.meter-label {
  overflow: hidden;
  color: var(--muted);
  font-size: 12px;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.meter-pct {
  color: var(--text);
  font-size: 12px;
  font-weight: 500;
  font-variant-numeric: tabular-nums;
  white-space: nowrap;
}

.meter-track {
  width: 100%;
  height: 6px;
  margin-top: 6px;
  overflow: hidden;
  border-radius: 999px;
  background: var(--track);
}

.meter-track i {
  display: block;
  height: 100%;
  border-radius: inherit;
  background: linear-gradient(90deg, var(--accent), color-mix(in srgb, var(--accent) 55%, var(--accent-2)));
  transition: width 0.5s ease;
}

.meter-foot {
  overflow: hidden;
  margin-top: 6px;
  color: var(--muted);
  font-size: 12px;
  font-variant-numeric: tabular-nums;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.net-row {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr)) repeat(2, minmax(5ch, auto));
  min-width: 0;
  align-items: center;
  justify-content: space-between;
  gap: 6px;
  margin-top: 12px;
  padding-top: 10px;
  border-top: 1px solid var(--line);
  font-size: 12px;
}

.net-item {
  display: inline-flex;
  min-width: 0;
  align-items: center;
  gap: 4px;
  color: var(--text);
  white-space: nowrap;
}

.net-item :deep(svg) {
  flex: 0 0 auto;
  color: var(--muted);
}

.net-item span {
  font-variant-numeric: tabular-nums;
}

.net-total {
  color: var(--muted);
}

.ping-row {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-top: 10px;
  color: var(--muted);
  font-size: 12px;
}

.ping-icon {
  flex: 0 0 auto;
  color: var(--muted);
}

.ping-list {
  display: flex;
  min-width: 0;
  flex-wrap: wrap;
  flex: 1;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}

.ping-item {
  overflow-wrap: anywhere;
}

.ping-item b {
  font-weight: 500;
}

.ping-item .is-empty {
  color: var(--bad);
  font-weight: 600;
}

@container (max-width: 460px) {
  .net-row {
    grid-template-columns: repeat(3, minmax(0, 1fr));
    row-gap: 8px;
  }
}

@container (max-width: 280px) {
  .net-row {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

@media (max-width: 380px) {
  .card-top {
    flex-direction: column;
    gap: 8px;
  }

  .card-status {
    align-items: flex-start;
  }
}
</style>
