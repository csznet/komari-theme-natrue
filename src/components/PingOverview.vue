<script setup lang="ts">
import type { NodeData } from '@/stores/nodes'
import { NButton, NCollapse, NCollapseItem, NEmpty, NSpin, NTag, NText } from 'naive-ui'
import { computed, defineAsyncComponent, ref, watch } from 'vue'
import PingChart from '@/components/PingChart.vue'
import { useAppStore } from '@/stores/app'
import { getRegionCode, getRegionDisplayName } from '@/utils/regionHelper'
import { getSharedRpc } from '@/utils/rpc'

const props = defineProps<{
  nodes: NodeData[]
}>()

const PingTaskChart = defineAsyncComponent(() => import('@/components/PingTaskChart.vue'))

const appStore = useAppStore()
const rpc = getSharedRpc()

// ==================== 类型定义 ====================

interface TaskInfo {
  id: number
  name: string
  interval: number
  loss: number
  p99?: number
  p50?: number
  avg?: number
  latest?: number
  type?: string
}

interface PingRecordsResponse {
  count: number
  records: unknown[]
  tasks?: TaskInfo[]
}

interface NodePingData {
  loading: boolean
  error: string | null
  tasks: TaskInfo[]
}

// ==================== 按节点展开模式 ====================

const pingDataMap = ref<Map<string, NodePingData>>(new Map())
const expandedNames = ref<string[]>([])

async function fetchNodePing(uuid: string) {
  const existing = pingDataMap.value.get(uuid)
  if (existing && (existing.loading || existing.tasks.length > 0))
    return

  pingDataMap.value.set(uuid, { loading: true, error: null, tasks: [] })

  try {
    const result = await rpc.getClient().call<PingRecordsResponse>('common:getRecords', {
      uuid,
      type: 'ping',
      hours: 1,
    })
    pingDataMap.value.set(uuid, {
      loading: false,
      error: null,
      tasks: result?.tasks || [],
    })
  }
  catch (err) {
    pingDataMap.value.set(uuid, {
      loading: false,
      error: err instanceof Error ? err.message : '获取失败',
      tasks: [],
    })
  }
}

function handleExpand(names: string[]) {
  expandedNames.value = names
  for (const uuid of names) {
    fetchNodePing(uuid)
  }
}

function expandAll() {
  const names = props.nodes.map(n => n.uuid)
  expandedNames.value = names
  for (const uuid of names) {
    fetchNodePing(uuid)
  }
}

function collapseAll() {
  expandedNames.value = []
}

function getNodePing(uuid: string): NodePingData | undefined {
  return pingDataMap.value.get(uuid)
}

function getLossColor(loss: number): 'success' | 'warning' | 'error' {
  if (loss === 0)
    return 'success'
  if (loss < 5)
    return 'warning'
  return 'error'
}

// ==================== 按监控任务归档模式 ====================

interface TaskGroup {
  taskName: string
  nodes: { uuid: string, name: string, taskId: number, latest?: number, loss: number, avg?: number }[]
}

const fetchAllLoading = ref(false)
const fetchAllError = ref<string | null>(null)
const taskGroups = ref<TaskGroup[]>([])
const taskExpandedNames = ref<string[]>([])

// 是否已获取全部（显示任务归档视图）
const showTaskView = computed(() => taskGroups.value.length > 0)

async function fetchAll() {
  fetchAllLoading.value = true
  fetchAllError.value = null

  try {
    const results = await Promise.allSettled(
      props.nodes.map(async (node) => {
        const result = await rpc.getClient().call<PingRecordsResponse>('common:getRecords', {
          uuid: node.uuid,
          type: 'ping',
          hours: 1,
        })
        return { uuid: node.uuid, name: node.name, tasks: result?.tasks || [] }
      }),
    )

    // 按 task name 归档
    const groupMap = new Map<string, TaskGroup>()

    for (const r of results) {
      if (r.status !== 'fulfilled')
        continue
      const { uuid, name, tasks } = r.value
      for (const task of tasks) {
        let group = groupMap.get(task.name)
        if (!group) {
          group = { taskName: task.name, nodes: [] }
          groupMap.set(task.name, group)
        }
        group.nodes.push({
          uuid,
          name,
          taskId: task.id,
          latest: task.latest,
          loss: task.loss,
          avg: task.avg,
        })
      }
    }

    taskGroups.value = Array.from(groupMap.values())
  }
  catch (err) {
    fetchAllError.value = err instanceof Error ? err.message : '获取失败'
    taskGroups.value = []
  }
  finally {
    fetchAllLoading.value = false
  }
}

function handleTaskExpand(names: string[]) {
  taskExpandedNames.value = names
}

function backToNodeView() {
  taskGroups.value = []
  taskExpandedNames.value = []
}

// 节点变化时清理
watch(() => props.nodes, () => {
  const validUuids = new Set(props.nodes.map(n => n.uuid))
  for (const key of pingDataMap.value.keys()) {
    if (!validUuids.has(key))
      pingDataMap.value.delete(key)
  }
  taskGroups.value = []
  taskExpandedNames.value = []
})
</script>

<template>
  <div class="flex flex-col gap-3">
    <!-- 任务归档视图 -->
    <template v-if="showTaskView">
      <div class="flex gap-2 items-center">
        <NButton size="small" tertiary @click="backToNodeView">
          返回节点视图
        </NButton>
        <NButton size="small" tertiary :loading="fetchAllLoading" @click="fetchAll">
          刷新
        </NButton>
      </div>

      <div v-if="fetchAllError" class="text-red-500 py-4 text-center">
        {{ fetchAllError }}
      </div>

      <NCollapse :expanded-names="taskExpandedNames" @update:expanded-names="handleTaskExpand">
        <NCollapseItem v-for="group in taskGroups" :key="group.taskName" :name="group.taskName">
          <template #header>
            <div class="flex gap-3 min-w-0 items-center">
              <span class="i-icon-park-outline-chart-line text-base flex-shrink-0" style="color: var(--n-text-color-2)" />
              <NText class="font-medium truncate">
                {{ group.taskName }}
              </NText>
            </div>
          </template>

          <template #header-extra>
            <div class="mr-2 flex flex-wrap gap-2 items-center">
              <NTag size="small" :bordered="false">
                <span :style="{ fontFamily: appStore.numberFontFamily }">{{ group.nodes.length }}</span>
                <span class="ml-1">节点</span>
              </NTag>
            </div>
          </template>

          <div class="pt-2">
            <PingTaskChart :task-name="group.taskName" :nodes="group.nodes" />
          </div>
        </NCollapseItem>
      </NCollapse>

      <div v-if="taskGroups.length === 0 && !fetchAllLoading" class="py-8">
        <NEmpty description="暂无延迟监控任务" />
      </div>
    </template>

    <!-- 节点视图 -->
    <template v-else>
      <div class="flex gap-2 items-center">
        <NButton size="small" tertiary @click="expandAll">
          全部展开
        </NButton>
        <NButton size="small" tertiary @click="collapseAll">
          全部收起
        </NButton>
        <NButton size="small" type="primary" :loading="fetchAllLoading" @click="fetchAll">
          获取全部
        </NButton>
      </div>

      <NSpin v-if="fetchAllLoading" class="py-8 w-full" />

      <NCollapse v-else :expanded-names="expandedNames" @update:expanded-names="handleExpand">
        <NCollapseItem v-for="node in nodes" :key="node.uuid" :name="node.uuid">
          <template #header>
            <div class="flex gap-3 min-w-0 items-center">
              <div
                class="rounded-full flex-shrink-0 h-2 w-2"
                :style="{ backgroundColor: node.online ? '#18a058' : '#d03050' }"
              />
              <img
                v-if="node.region"
                :src="`/images/flags/${getRegionCode(node.region)}.svg`"
                class="flex-shrink-0 h-4 w-5 object-contain"
                :alt="getRegionDisplayName(node.region)"
              >
              <NText class="font-medium truncate">
                {{ node.name }}
              </NText>
            </div>
          </template>

          <template #header-extra>
            <div class="mr-2 flex flex-wrap gap-2 items-center">
              <template v-if="getNodePing(node.uuid)?.tasks?.length">
                <NTag v-for="task in getNodePing(node.uuid)!.tasks" :key="task.id" size="small" :bordered="false">
                  <span class="font-medium">{{ task.name }}</span>
                  <span class="mx-1 opacity-40">|</span>
                  <span :style="{ fontFamily: appStore.numberFontFamily }">
                    {{ task.latest !== undefined ? `${Math.round(task.latest)}ms` : '-' }}
                  </span>
                  <span class="mx-1 opacity-40">|</span>
                  <NText :type="getLossColor(task.loss)">
                    <span :style="{ fontFamily: appStore.numberFontFamily }">{{ task.loss.toFixed(1) }}%</span>
                  </NText>
                </NTag>
              </template>
              <NSpin v-else-if="getNodePing(node.uuid)?.loading" :size="14" />
            </div>
          </template>

          <div class="pt-2">
            <NSpin v-if="getNodePing(node.uuid)?.loading" class="py-8 w-full" />
            <div v-else-if="getNodePing(node.uuid)?.error" class="text-sm text-red-500 py-4 text-center">
              {{ getNodePing(node.uuid)!.error }}
            </div>
            <div v-else-if="getNodePing(node.uuid)?.tasks?.length === 0" class="py-4">
              <NEmpty description="暂无延迟监控任务" size="small" />
            </div>
            <PingChart v-else :uuid="node.uuid" />
          </div>
        </NCollapseItem>
      </NCollapse>

      <div v-if="nodes.length === 0" class="py-8">
        <NEmpty description="暂无节点" />
      </div>
    </template>
  </div>
</template>
