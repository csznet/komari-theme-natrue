<script setup lang="ts">
import dayjs from 'dayjs'
import { NButton, NEmpty, NSpin, NSwitch, NTooltip } from 'naive-ui'
import { computed, onMounted, ref, shallowRef, watch } from 'vue'
import VChart from 'vue-echarts'
import { useAppStore } from '@/stores/app'
import { cutPeakValues, interpolateNullsLinear } from '@/utils/recordHelper'
import { getSharedRpc } from '@/utils/rpc'
import '@/utils/echarts'

interface NodeEntry {
  uuid: string
  name: string
  taskId: number
}

const props = defineProps<{
  taskName: string
  nodes: NodeEntry[]
}>()

const appStore = useAppStore()
const isDark = computed(() => appStore.isDark)
const rpc = getSharedRpc()

const chartThemeColors = computed(() => ({
  text: isDark.value ? 'rgba(255, 255, 255, 0.85)' : 'rgba(0, 0, 0, 0.85)',
  textSecondary: isDark.value ? 'rgba(255, 255, 255, 0.55)' : 'rgba(0, 0, 0, 0.55)',
  textTertiary: isDark.value ? 'rgba(255, 255, 255, 0.35)' : 'rgba(0, 0, 0, 0.35)',
  borderColor: isDark.value ? 'rgba(255, 255, 255, 0.1)' : 'rgba(0, 0, 0, 0.1)',
  splitLineColor: isDark.value ? 'rgba(255, 255, 255, 0.06)' : 'rgba(0, 0, 0, 0.06)',
  tooltipBg: isDark.value ? 'rgba(40, 40, 40, 0.95)' : 'rgba(255, 255, 255, 0.98)',
  tooltipShadow: isDark.value ? 'rgba(0, 0, 0, 0.4)' : 'rgba(0, 0, 0, 0.12)',
  crosshairColor: isDark.value ? 'rgba(255, 255, 255, 0.15)' : 'rgba(0, 0, 0, 0.1)',
}))

const chartColors = [
  '#FF6B6B',
  '#4ECDC4',
  '#A78BFA',
  '#60A5FA',
  '#FFB347',
  '#F472B6',
  '#34D399',
  '#FB923C',
  '#818CF8',
  '#F87171',
  '#38BDF8',
  '#FBBF24',
]

const maxPingRecordPreserveTime = computed(() => appStore.publicSettings?.ping_record_preserve_time || 168)

const presetViews = [
  { label: '15 分钟', hours: 0.25 },
  { label: '1 小时', hours: 1 },
  { label: '6 小时', hours: 6 },
  { label: '12 小时', hours: 12 },
  { label: '1 天', hours: 24 },
]

const availableViews = computed(() => {
  const views: { label: string, hours: number }[] = []
  const maxHours = maxPingRecordPreserveTime.value
  for (const v of presetViews) {
    if (maxHours >= v.hours)
      views.push(v)
  }
  const maxPreset = presetViews[presetViews.length - 1]
  if (maxPreset && maxHours > maxPreset.hours) {
    const label = maxHours % 24 === 0 ? `${Math.floor(maxHours / 24)} 天` : `${maxHours} 小时`
    views.push({ label, hours: maxHours })
  }
  else if (maxHours > 1 && !presetViews.some(v => v.hours === maxHours)) {
    const label = maxHours % 24 === 0 ? `${Math.floor(maxHours / 24)} 天` : `${maxHours} 小时`
    views.push({ label, hours: maxHours })
  }
  return views
})

const selectedView = ref<string>('')
const selectedHours = computed(() => {
  const view = availableViews.value.find(v => v.label === selectedView.value)
  return view?.hours || 1
})

watch(availableViews, (views) => {
  const firstView = views[0]
  if (firstView && !selectedView.value)
    selectedView.value = firstView.label
}, { immediate: true })

// ==================== 类型定义 ====================

interface PingRecord {
  client: string
  task_id: number
  time: string
  value: number
}

interface PingRecordsResponse {
  count: number
  records: PingRecord[]
  tasks?: unknown[]
}

// ==================== 数据状态 ====================

// key = uuid, value = records for this node's specific task
const nodeRecordsMap = shallowRef<Map<string, PingRecord[]>>(new Map())
const loading = ref(false)
const error = ref<string | null>(null)
const cutPeak = ref(false)

const selectedNodeUuids = ref<string[]>([])

const chartMargin = { top: 12, right: 24, bottom: 52, left: 56 }

// ==================== 数据获取 ====================

async function fetchAllRecords() {
  if (!props.nodes.length)
    return

  loading.value = true
  error.value = null

  try {
    const results = await Promise.allSettled(
      props.nodes.map(async (entry) => {
        const result = await rpc.getClient().call<PingRecordsResponse>('common:getRecords', {
          uuid: entry.uuid,
          type: 'ping',
          hours: selectedHours.value,
        })
        const records = (result?.records || [])
          .filter(r => r.task_id === entry.taskId)
          .sort((a, b) => dayjs(a.time).valueOf() - dayjs(b.time).valueOf())
        return { uuid: entry.uuid, records }
      }),
    )

    const newMap = new Map<string, PingRecord[]>()
    for (const r of results) {
      if (r.status === 'fulfilled')
        newMap.set(r.value.uuid, r.value.records)
    }
    nodeRecordsMap.value = newMap

    if (selectedNodeUuids.value.length === 0)
      selectedNodeUuids.value = props.nodes.map(n => n.uuid)
  }
  catch (err) {
    error.value = err instanceof Error ? err.message : '获取数据失败'
    nodeRecordsMap.value = new Map()
  }
  finally {
    loading.value = false
  }
}

// ==================== 数据处理 ====================

const mergedData = computed(() => {
  const map = nodeRecordsMap.value
  if (!map.size)
    return []

  // Collect all records, use uuid as "series key"
  const allRecords: { uuid: string, time: number, value: number | null }[] = []
  for (const [uuid, records] of map) {
    for (const rec of records) {
      allRecords.push({
        uuid,
        time: dayjs(rec.time).valueOf(),
        value: rec.value < 0 ? null : rec.value,
      })
    }
  }

  if (!allRecords.length)
    return []

  // Find a reasonable tolerance for grouping timestamps
  const toleranceMs = 6000

  const grouped: Map<number, Record<string, unknown>> = new Map()
  const anchors: number[] = []

  for (const rec of allRecords) {
    let anchor: number | null = null
    for (const a of anchors) {
      if (Math.abs(a - rec.time) <= toleranceMs) {
        anchor = a
        break
      }
    }

    const useTs = anchor ?? rec.time
    if (!grouped.has(useTs)) {
      grouped.set(useTs, { time: dayjs(useTs).toISOString() })
      if (anchor === null)
        anchors.push(useTs)
    }

    grouped.get(useTs)![rec.uuid] = rec.value
  }

  const merged = Array.from(grouped.values()).sort(
    (a, b) => dayjs(a.time as string).valueOf() - dayjs(b.time as string).valueOf(),
  )

  const hours = selectedHours.value
  const lastItem = merged[merged.length - 1]
  const lastTs = lastItem ? dayjs(lastItem.time as string).valueOf() : dayjs().valueOf()
  const fromTs = lastTs - hours * 3600_000

  let startIdx = 0
  for (let i = 0; i < merged.length; i++) {
    const item = merged[i]
    if (!item)
      continue
    if (dayjs(item.time as string).valueOf() >= fromTs) {
      startIdx = Math.max(0, i - 1)
      break
    }
  }

  return merged.slice(startIdx)
})

const chartData = computed(() => {
  let data = mergedData.value
  const selectedKeys = selectedNodeUuids.value

  if (selectedKeys.length === 0)
    return []

  if (cutPeak.value)
    data = cutPeakValues(data, selectedKeys)

  if (selectedKeys.length > 0 && data.length > 0) {
    data = interpolateNullsLinear(data, selectedKeys, {
      maxGapMultiplier: 6,
      minCapMs: 2 * 60_000,
      maxCapMs: 30 * 60_000,
    })
  }

  return data
})

// ==================== 工具函数 ====================

function formatTime(time: string, showDate: boolean): string {
  const date = dayjs(time)
  return showDate ? date.format('M/D HH:mm') : date.format('HH:mm')
}

function formatTimeForTooltip(time: string, hours: number): string {
  const date = dayjs(time)
  return hours < 24 ? date.format('HH:mm:ss') : date.format('MM/DD HH:mm')
}

const showDateInAxis = computed(() => selectedHours.value >= 24)

function getNodeColor(uuid: string): string {
  const idx = props.nodes.findIndex(n => n.uuid === uuid)
  return chartColors[Math.max(0, idx % chartColors.length)]!
}

// ==================== 节点选择 ====================

function toggleNode(uuid: string) {
  if (selectedNodeUuids.value.includes(uuid))
    selectedNodeUuids.value = selectedNodeUuids.value.filter(id => id !== uuid)
  else
    selectedNodeUuids.value = [...selectedNodeUuids.value, uuid]
}

function showAll() {
  selectedNodeUuids.value = props.nodes.map(n => n.uuid)
}

function hideAll() {
  selectedNodeUuids.value = []
}

// ==================== 图表配置 ====================

const chartOption = computed(() => {
  const selectedNodes = props.nodes.filter(n => selectedNodeUuids.value.includes(n.uuid))
  const data = chartData.value

  const colorMap = new Map<string, string>()
  props.nodes.forEach((n, idx) => {
    colorMap.set(n.uuid, chartColors[Math.max(0, idx % chartColors.length)]!)
  })

  const series = selectedNodes.map((node) => {
    const color = getNodeColor(node.uuid)
    return {
      name: node.name,
      type: 'line' as const,
      data: data.map(d => d[node.uuid] as number | null ?? null),
      smooth: cutPeak.value ? 0.6 : 0.4,
      showSymbol: false,
      connectNulls: false,
      lineStyle: { width: 2.5, color, cap: 'round' as const },
      itemStyle: { color },
    }
  })

  const hours = selectedHours.value

  return {
    animation: false,
    color: props.nodes.map((_, idx) => chartColors[Math.max(0, idx % chartColors.length)]!),
    tooltip: {
      trigger: 'axis' as const,
      confine: false,
      backgroundColor: chartThemeColors.value.tooltipBg,
      borderColor: 'transparent',
      borderWidth: 0,
      borderRadius: 8,
      padding: [10, 14],
      boxShadow: `0 4px 16px ${chartThemeColors.value.tooltipShadow}`,
      textStyle: {
        color: chartThemeColors.value.text,
        fontSize: 13,
        lineHeight: 20,
      },
      extraCssText: 'box-shadow: none; backdrop-filter: blur(8px);',
      axisPointer: {
        type: 'cross' as const,
        crossStyle: { color: chartThemeColors.value.textTertiary },
        lineStyle: { color: chartThemeColors.value.crosshairColor, width: 1, type: 'dashed' as const },
        shadowStyle: { color: chartThemeColors.value.crosshairColor },
      },
      formatter: (params: unknown) => {
        const p = params as Array<{ seriesName: string, value: number | null, dataIndex: number }>
        if (!p.length)
          return ''
        const firstParam = p[0]
        if (!firstParam)
          return ''
        const rowData = data[firstParam.dataIndex]
        if (!rowData)
          return ''

        const time = rowData.time as string
        const timeStr = formatTimeForTooltip(time, hours)
        let html = `<div style="font-weight:600;margin-bottom:6px;color:${chartThemeColors.value.textSecondary}">${timeStr}</div>`
        html += '<div style="display:flex;flex-direction:column;gap:4px">'

        const sortedParams = [...p].sort((a, b) => (a.value ?? 0) - (b.value ?? 0))
        for (const item of sortedParams) {
          if (item.value !== null && item.value !== undefined) {
            const node = props.nodes.find(n => n.name === item.seriesName)
            const color = node ? colorMap.get(node.uuid) || chartColors[0] : chartColors[0]
            const dot = `<span style="display:inline-block;width:8px;height:8px;border-radius:50%;background:${color};margin-right:8px;flex-shrink:0"></span>`
            html += `<div style="display:flex;align-items:center">${dot}<span style="flex:1;overflow:hidden;text-overflow:ellipsis;white-space:nowrap">${item.seriesName}</span><span style="margin-left:16px;font-weight:600;font-variant-numeric:tabular-nums">${Math.round(item.value)} ms</span></div>`
          }
        }
        html += '</div>'
        return html
      },
    },
    legend: {
      type: 'scroll',
      bottom: 4,
      itemWidth: 12,
      itemHeight: 12,
      itemGap: 16,
      icon: 'roundRect',
      textStyle: { fontSize: 11, color: chartThemeColors.value.textSecondary },
      data: selectedNodes.map(n => n.name),
    },
    grid: chartMargin,
    xAxis: {
      type: 'category',
      data: data.map(d => formatTime(d.time as string, showDateInAxis.value)),
      axisLabel: { fontSize: 11, color: chartThemeColors.value.textSecondary, margin: 12 },
      axisLine: { show: true, lineStyle: { color: chartThemeColors.value.borderColor, width: 1 } },
      axisTick: { show: false },
      boundaryGap: false,
    },
    yAxis: {
      type: 'value',
      name: '延迟 (ms)',
      nameTextStyle: { color: chartThemeColors.value.textSecondary, padding: [0, 40, 0, 0] },
      axisLabel: { fontSize: 11, color: chartThemeColors.value.textSecondary, formatter: '{value}' },
      axisLine: { show: false },
      axisTick: { show: false },
      splitLine: {
        lineStyle: { color: chartThemeColors.value.splitLineColor, type: 'dashed' as const },
      },
    },
    series,
  }
})

// ==================== 生命周期 ====================

watch(selectedView, () => {
  selectedNodeUuids.value = []
  fetchAllRecords()
})

watch(() => props.nodes, () => {
  nodeRecordsMap.value = new Map()
  selectedNodeUuids.value = []
  fetchAllRecords()
})

onMounted(() => {
  const firstView = availableViews.value[0]
  if (firstView && !selectedView.value)
    selectedView.value = firstView.label
  fetchAllRecords()
})
</script>

<template>
  <div class="flex flex-col gap-4">
    <!-- 时间选择器 -->
    <div class="flex flex-wrap gap-2 justify-center">
      <NButton
        v-for="view in availableViews"
        :key="view.label"
        :type="selectedView === view.label ? 'primary' : 'default'"
        size="small"
        @click="selectedView = view.label"
      >
        {{ view.label }}
      </NButton>
    </div>

    <NSpin :show="loading" content-class="flex flex-col gap-4">
      <div v-if="error" class="text-red-500 py-8 text-center">
        {{ error }}
      </div>
      <div v-else-if="nodeRecordsMap.size === 0 && !loading" class="py-8">
        <NEmpty description="暂无延迟数据" />
      </div>

      <template v-else>
        <!-- 节点选择标签 -->
        <div class="flex flex-wrap gap-2">
          <div
            v-for="node in nodes"
            :key="node.uuid"
            class="text-sm px-3 py-1.5 rounded-md cursor-pointer select-none transition-all"
            :class="selectedNodeUuids.includes(node.uuid) ? '' : 'opacity-40'"
            :style="{
              backgroundColor: `${getNodeColor(node.uuid)}18`,
              border: `1px solid ${getNodeColor(node.uuid)}40`,
              color: getNodeColor(node.uuid),
            }"
            @click="toggleNode(node.uuid)"
          >
            {{ node.name }}
          </div>
        </div>

        <!-- 控制栏 -->
        <div class="flex flex-wrap gap-4 items-center">
          <div class="flex gap-2 items-center">
            <NSwitch v-model:value="cutPeak" size="small" />
            <span class="text-sm">裁剪峰值</span>
            <NTooltip>
              <template #trigger>
                <span class="i-carbon-information text-sm opacity-50 cursor-help transition-opacity hover:opacity-100" style="color: var(--n-text-color-3)" />
              </template>
              <span>使用 EWMA 算法平滑数据并过滤突变值</span>
            </NTooltip>
          </div>
          <div class="flex gap-2 items-center">
            <NButton size="small" tertiary @click="showAll">
              全选
            </NButton>
            <NButton size="small" tertiary @click="hideAll">
              全不选
            </NButton>
          </div>
        </div>

        <!-- 图表 -->
        <div class="h-80">
          <VChart :option="chartOption" autoresize />
        </div>
      </template>
    </NSpin>
  </div>
</template>
