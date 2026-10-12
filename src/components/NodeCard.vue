<script setup lang="ts">
import type { NodeData } from '@/stores/nodes'
import { Icon } from '@iconify/vue'
import { computed } from 'vue'
import { Badge } from '@/components/ui/badge'
import { CardX } from '@/components/ui/card-x'
import { DataTooltip } from '@/components/ui/data-tooltip'
import { ProgressThin } from '@/components/ui/progress-thin'
import { useBackgroundSurface } from '@/composables/useBackgroundSurface'
import { useNodeFormatters } from '@/composables/useNodeFormatters'
import { useNodePingDisplay } from '@/composables/useNodePingDisplay'
import { useAppStore } from '@/stores/app'
import { formatDateTime, getStatus } from '@/utils/helper'
import { formatOfflineDuration, getCustomTags, getDiskPercentage, getMemPercentage, getPriceTags, getRemainingTimeTagClass, getTrafficUsed, getTrafficUsedPercentage, hasRegion, showTrafficProgress } from '@/utils/nodeHelpers'
import { getOSImage, getOSName } from '@/utils/osImageHelper'
import { getFlagSrc, getRegionDisplayName } from '@/utils/regionHelper'

const props = defineProps<{ node: NodeData }>()

const emit = defineEmits<{
  click: []
  pingClick: [node: NodeData]
}>()

const appStore = useAppStore()
const { pickSurfaceClass } = useBackgroundSurface()
const { formatBytes, formatBytesPerSecond, formatUptime } = useNodeFormatters()

const offlineTime = computed(() => {
  if (props.node.last_seen_ago === null)
    return '从未上报'
  if (typeof props.node.last_seen_ago === 'number')
    return formatOfflineDuration(props.node.last_seen_ago)
  return formatDateTime(props.node.time)
})
const expiredDate = computed(() => formatDateTime(props.node.expired_at, 'YYYY-MM-DD'))

const cpuStatus = computed(() => getStatus(props.node.cpu ?? 0))
const memPercentage = computed(() => getMemPercentage(props.node))
const memStatus = computed(() => getStatus(memPercentage.value))
const diskPercentage = computed(() => getDiskPercentage(props.node))
const diskStatus = computed(() => getStatus(diskPercentage.value))

const {
  latencyRenderBars,
  lossRenderBars,
  latencyDisplay,
  lossDisplay,
  latencyPanelTooltip,
  lossPanelTooltip,
  topPingNetworks,
} = useNodePingDisplay(() => props.node.uuid)

const trafficUsedPercentage = computed(() => getTrafficUsedPercentage(props.node))
const trafficUsed = computed(() => getTrafficUsed(props.node))
const priceTags = computed(() => getPriceTags(props.node, appStore.lang))
const remainingTimeTagClass = computed(() => getRemainingTimeTagClass(props.node))
const customTags = computed(() => getCustomTags(props.node))

function openPingDialog() {
  emit('pingClick', props.node)
}
</script>

<template>
  <CardX
    hoverable
    class="node-card h-full w-full cursor-pointer border-none shadow-[0_0_0_1px] shadow-transparent transition-all duration-200 rounded-md bg-background/60 hover:bg-background hover:shadow-emerald-600/10 hover:shadow-[0_0_20px,0_0_0_1px] hover:-translate-y-0.5 hover:z-1"
    :class="[pickSurfaceClass('', 'backdrop-blur-sm'), !props.node.online && 'shadow-[0_0_0_1px] !shadow-red-600/20']"
    @click="emit('click')"
  >
    <template #header>
      <div class="flex gap-2 min-w-0 items-center">
        <div class="size-2 rounded-full relative" :class="[props.node.online ? 'bg-emerald-600' : 'bg-red-600']">
          <div
            class="animate-ping absolute inset-0 rounded-full opacity-50"
            :class="[props.node.online ? 'bg-emerald-600' : 'bg-red-600']"
          />
        </div>
        <div class="text-md font-bold flex-1 min-w-0 truncate">
          {{ props.node.name }}
        </div>
      </div>
    </template>

    <template #header-extra>
      <div class="flex gap-2 items-center">
        <img :src="getOSImage(props.node.os)" :alt="getOSName(props.node.os)" class="size-4">
        <img
          v-if="hasRegion(props.node.region)" :src="getFlagSrc(props.node.region)"
          :alt="getRegionDisplayName(props.node.region)" class="size-5 shrink-0"
        >
      </div>
    </template>

    <template #default>
      <div class="flex flex-col gap-3">
        <div class="gap-x-3 gap-y-1 grid grid-cols-2">
          <!-- CPU -->
          <div class="flex flex-col gap-1">
            <div class="w-full text-xs flex flex-row justify-between">
              <span class="text-muted-foreground">
                CPU
              </span>
              <span>{{ (props.node.cpu ?? 0).toFixed(1) }}%</span>
            </div>
            <ProgressThin :percentage="props.node.cpu ?? 0" :status="cpuStatus" :height="4" />
            <div class="text-[11px] text-muted-foreground truncate">
              {{ props.node.load.toFixed(2) ?? 0 }}, {{ props.node.load5.toFixed(2) ?? 0 }}, {{
                props.node.load15.toFixed(2) ?? 0 }}
            </div>
          </div>

          <!-- 内存 -->
          <div class="flex flex-col gap-1">
            <div class="w-full text-xs flex flex-row justify-between">
              <span class="text-muted-foreground">
                内存
              </span>
              <span>{{ memPercentage.toFixed(1) }}%</span>
            </div>
            <ProgressThin :percentage="memPercentage" :status="memStatus" :height="4" />
            <DataTooltip placement="top" class="block" :content-class="[!props.node.swap && '!hidden']">
              <div class="text-[11px] text-muted-foreground truncate">
                {{ formatBytes(props.node.ram ?? 0) }} / {{ formatBytes(props.node.mem_total ?? 0) }}
              </div>
              <template #content>
                <div class="flex items-center justify-between gap-3 whitespace-nowrap">
                  <span class="text-background/70">Swap</span>
                  <span>{{ formatBytes(props.node.swap ?? 0) }}</span>
                </div>
              </template>
            </DataTooltip>
          </div>

          <!-- 硬盘 -->
          <div class="flex flex-col gap-1">
            <div class="w-full text-xs flex flex-row justify-between">
              <span class="text-muted-foreground">
                硬盘
              </span>
              <span>{{ diskPercentage.toFixed(1) }}%</span>
            </div>
            <ProgressThin :percentage="diskPercentage" :status="diskStatus" :height="4" />
            <div class="text-[11px] text-muted-foreground truncate">
              {{ formatBytes(props.node.disk ?? 0) }} / {{ formatBytes(props.node.disk_total ?? 0) }}
            </div>
          </div>

          <!-- 流量进度条 -->
          <div class="flex flex-col gap-1">
            <div class="w-full text-xs flex flex-row justify-between">
              <span class="text-muted-foreground">
                流量
              </span>
              <span>{{ trafficUsedPercentage.toFixed(1) }}%</span>
            </div>
            <ProgressThin :percentage="trafficUsedPercentage" status="success" :height="4" />
            <DataTooltip placement="top" class="block">
              <div class="whitespace-pre-wrap text-[11px] text-muted-foreground truncate">
                {{ formatBytes(trafficUsed) }} /
                <template v-if="showTrafficProgress(props.node)">
                  {{ formatBytes(props.node.traffic_limit) }}
                </template>
                <template v-else>
                  ∞
                </template>
              </div>
              <template #content>
                <div class="flex items-center justify-between gap-3 whitespace-nowrap">
                  <div class="text-[11px] flex flex-col">
                    <div class="flex flex-row items-center gap-1">
                      <Icon icon="tabler:chevron-up" width="12" height="12" />
                      {{ formatBytes(props.node.net_total_up ?? 0) }}
                    </div>
                    <div class="flex flex-row items-center gap-1">
                      <Icon icon="tabler:chevron-down" width="12" height="12" />
                      {{ formatBytes(props.node.net_total_down ?? 0) }}
                    </div>
                  </div>
                </div>
              </template>
            </DataTooltip>
          </div>
        </div>
        <div class="relative text-[11px] text-muted-foreground">
          <div
            v-if="!props.node.online"
            class="absolute inset-0 z-10 flex flex-col items-center justify-center space-y-1"
          >
            <span class="text-sm text-red-600">离线</span>
            <div :title="formatDateTime(props.node.time)">
              {{ offlineTime }}
            </div>
          </div>
          <div
            class="grid grid-cols-2 gap-1.5"
            :class="[!props.node.online && 'blur-xs opacity-60 pointer-events-none']"
          >
            <!-- 速率 -->
            <div class="col-span-2 rounded-md nc-tile bg-muted px-2 py-1.5">
              <div class="flex items-center justify-between gap-2">
                <span class="shrink-0 text-[10px] font-medium tracking-wider text-muted-foreground">速率</span>
                <div class="grid min-w-0 flex-1 grid-cols-2 gap-2">
                  <div class="flex flex-row items-center justify-end gap-0.5 text-green-600">
                    <Icon icon="tabler:chevron-up" width="12" height="12" />
                    {{ formatBytesPerSecond(props.node.net_out ?? 0) }}
                  </div>
                  <div class="flex flex-row items-center justify-end gap-0.5 text-blue-600">
                    <Icon icon="tabler:chevron-down" width="12" height="12" />
                    {{ formatBytesPerSecond(props.node.net_in ?? 0) }}
                  </div>
                </div>
              </div>
            </div>
            <!-- 在线 -->
            <div class="min-h-10 rounded-md nc-tile bg-muted px-2 py-1.5">
              <div class="flex items-start justify-between gap-1.5">
                <span class="shrink-0 text-[10px] font-medium tracking-wider text-muted-foreground">在线</span>
                <span class="min-w-0 truncate text-right font-medium text-foreground/85">
                  {{ props.node.uptime > 0 ? formatUptime(props.node.uptime) : '—' }}
                </span>
              </div>
            </div>
            <!-- 费用 -->
            <div class="min-h-10 rounded-md nc-tile bg-muted px-2 py-1.5">
              <div class="flex items-start justify-between gap-1.5">
                <span class="shrink-0 text-[10px] font-medium tracking-wider text-muted-foreground">费用</span>
                <DataTooltip
                  placement="left" :content="expiredDate" class="min-w-0"
                  content-class="whitespace-nowrap right-0 mr-0"
                >
                  <span class="flex min-w-0 flex-row flex-wrap justify-end gap-x-1 leading-tight">
                    <template v-for="(tag, index) in priceTags" :key="tag.text">
                      <span class="inline-flex flex-row items-center gap-1">
                        <span :class="tag.highlight ? remainingTimeTagClass : ''">{{ tag.text }}</span>
                      </span>
                      <span v-if="index < priceTags.length - 1">·</span>
                    </template>
                  </span>
                </DataTooltip>
              </div>
            </div>
            <!-- 三网 -->
            <div class="col-span-2 rounded-md nc-tile bg-muted px-2 py-1.5">
              <div class="flex items-center justify-between gap-2">
                <span class="shrink-0 text-[10px] font-medium tracking-wider text-muted-foreground">三网</span>
                <div v-if="topPingNetworks.length > 0" class="flex min-w-0 flex-row">
                  <DataTooltip
                    v-for="(net, index) in topPingNetworks" :key="net.name" placement="top"
                    :content="`${net.name}\n${net.latency}`" content-class="whitespace-pre-wrap w-max px-1.5 !leading-[1.2] text-[11px]"
                  >
                    <div class="truncate">
                      <span v-if="index" class="mx-1">·</span>
                      <span :class="net.toneClass">{{ net.latency }}</span>
                    </div>
                  </DataTooltip>
                </div>
                <div v-else class="min-w-0 truncate">
                  N/A
                </div>
              </div>
            </div>
            <!-- 延迟 -->
            <div
              role="button" tabindex="0"
              class="group/panel flex cursor-pointer flex-col gap-1.5 rounded-md nc-tile bg-muted px-2 py-1.5 text-left transition-colors hover:bg-foreground/[0.06] focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring"
              :title="latencyPanelTooltip" :aria-label="`${props.node.name} 延迟`" @click.stop="openPingDialog"
              @keydown.enter.stop.prevent="openPingDialog" @keydown.space.stop.prevent="openPingDialog"
            >
              <div class="flex items-center justify-between gap-2 text-[11px] leading-none">
                <span class="shrink-0 text-[10px] font-medium tracking-wider text-muted-foreground">延迟</span>
                <span class="font-medium text-foreground/85">{{ latencyDisplay }}</span>
              </div>
              <div
                class="grid h-2 items-end gap-[1px]"
                :style="{ gridTemplateColumns: `repeat(${latencyRenderBars.length}, minmax(0, 1fr))` }"
              >
                <DataTooltip
                  v-for="bar in latencyRenderBars" :key="bar.key" placement="top" :content="bar.tooltip"
                  class="h-full w-full" content-class="whitespace-pre-wrap w-max px-1.5 !leading-[1.2] text-[11px]"
                >
                  <span
                    class="block h-full w-full rounded-[1px] transition-transform duration-150 group-hover/data-tooltip:scale-y-200"
                    :class="bar.className"
                  />
                </DataTooltip>
              </div>
            </div>
            <!-- 丢包 -->
            <div
              role="button" tabindex="0"
              class="group/panel flex cursor-pointer flex-col gap-1.5 rounded-md nc-tile bg-muted px-2 py-1.5 text-left transition-colors hover:bg-foreground/[0.06] focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring"
              :title="lossPanelTooltip" :aria-label="`${props.node.name} 丢包`" @click.stop="openPingDialog"
              @keydown.enter.stop.prevent="openPingDialog" @keydown.space.stop.prevent="openPingDialog"
            >
              <div class="flex items-center justify-between gap-2 text-[11px] leading-none">
                <span class="shrink-0 text-[10px] font-medium tracking-wider text-muted-foreground">丢包</span>
                <span class="font-medium text-foreground/85">{{ lossDisplay }}</span>
              </div>
              <div
                class="grid h-2 items-end gap-[1px]"
                :style="{ gridTemplateColumns: `repeat(${lossRenderBars.length}, minmax(0, 1fr))` }"
              >
                <DataTooltip
                  v-for="bar in lossRenderBars" :key="bar.key" placement="top" :content="bar.tooltip"
                  class="h-full w-full" content-class="whitespace-pre-wrap w-max px-1.5 !leading-[1.2] text-[11px]"
                >
                  <span
                    class="block h-full w-full rounded-[1px] transition-transform duration-150 group-hover/data-tooltip:scale-y-200"
                    :class="bar.className"
                  />
                </DataTooltip>
              </div>
            </div>
          </div>
        </div>
        <div v-if="props.node.public_remark" class="text-[11px] text-muted-foreground/80 truncate" :title="props.node.public_remark">
          {{ props.node.public_remark }}
        </div>
        <div v-if="customTags.length > 0" class="flex shrink-0 flex-wrap gap-1 items-center">
          <Badge
            v-for="(tag, index) in customTags" :key="index" variant="outline"
            class="!text-[11px] rounded text-muted-foreground border-muted-foreground/10 px-1.5"
          >
            {{ tag }}
          </Badge>
        </div>
      </div>
    </template>
  </CardX>
</template>

<style scoped>
.node-card {
  position: relative;
}
</style>
