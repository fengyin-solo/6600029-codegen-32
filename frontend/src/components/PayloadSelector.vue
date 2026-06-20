<script setup lang="ts">
import { useDroneStore } from '../store/drone';
import { computed } from 'vue';

const store = useDroneStore();

function formatTime(seconds: number): string {
  const m = Math.floor(seconds / 60);
  const s = Math.floor(seconds % 60);
  return `${m}:${s.toString().padStart(2, '0')}`;
}

function batteryColor(pct: number): string {
  if (pct < 50) return '#22c55e';
  if (pct < 80) return '#eab308';
  return '#ef4444';
}

const statsDiff = computed(() => {
  if (!store.currentPlan || store.waypoints.length < 2) return null;
  const baseTime = store.totalDistance / 10;
  const baseBattery = (baseTime / 60 * store.droneConfig.consumptionRate / store.droneConfig.batteryCapacity) * 100;
  return {
    timeDelta: store.estimatedTime - baseTime,
    batteryDelta: store.batteryPercent - baseBattery,
    distanceDelta: 0,
  };
});

function formatDelta(val: number, unit = ''): string {
  if (Math.abs(val) < 0.01) return `—`;
  const sign = val > 0 ? '+' : '';
  return `${sign}${val.toFixed(1)}${unit}`;
}
</script>

<template>
  <div class="bg-slate-800 rounded-lg p-4 space-y-4">
    <h3 class="text-sm font-bold text-slate-200 border-b border-slate-700 pb-2">
      🔧 设备与载荷评估
    </h3>

    <!-- Drone Model Selector -->
    <div>
      <label class="block text-xs font-semibold text-slate-300 mb-2">无人机型号</label>
      <div class="grid grid-cols-1 gap-2">
        <label
          v-for="model in store.droneModels"
          :key="model.id"
          class="cursor-pointer block"
        >
          <input
            type="radio"
            :value="model.id"
            v-model="store.selectedDroneModelId"
            @change="store.applyDroneModel(model.id)"
            class="hidden peer"
          />
          <div
            class="flex items-center gap-2 p-2 rounded border text-xs transition"
            :class="
              store.selectedDroneModelId === model.id
                ? 'bg-sky-900/40 border-sky-600 text-sky-200'
                : 'bg-slate-900 border-slate-700 text-slate-400 hover:border-slate-600'
            "
          >
            <span class="text-lg">{{ model.icon }}</span>
            <div class="flex-1">
              <div class="font-semibold">{{ model.name }}</div>
              <div class="text-[10px] opacity-75">
                最大载荷 {{ model.maxPayloadWeight }}kg · 电池 {{ model.baseConfig.batteryCapacity }}mAh
              </div>
            </div>
          </div>
        </label>
      </div>
    </div>

    <!-- Payload Selector -->
    <div>
      <label class="block text-xs font-semibold text-slate-300 mb-2">挂载载荷</label>
      <div class="grid grid-cols-2 gap-2">
        <label
          v-for="payload in store.availablePayloads"
          :key="payload.id"
          class="cursor-pointer block"
        >
          <input
            type="radio"
            :value="payload.id"
            v-model="store.selectedPayloadId"
            @change="store.applyPayload(payload.id)"
            class="hidden peer"
          />
          <div
            class="p-2 rounded border text-xs transition text-center"
            :class="
              store.selectedPayloadId === payload.id
                ? 'bg-purple-900/40 border-purple-500 text-purple-200'
                : 'bg-slate-900 border-slate-700 text-slate-400 hover:border-slate-600'
            "
          >
            <div class="text-xl mb-0.5">{{ payload.icon }}</div>
            <div class="font-semibold text-[11px]">{{ payload.name }}</div>
            <div class="text-[9px] opacity-75 mt-0.5">
              {{ payload.weight }}kg
            </div>
          </div>
        </label>
      </div>
      <div v-if="store.selectedPayload" class="mt-2 p-2 bg-slate-900/60 rounded text-[10px] text-slate-400">
        <div class="font-medium text-slate-300 mb-0.5">{{ store.selectedPayload.name }} · {{ store.selectedPayload.icon }}</div>
        <div>{{ store.selectedPayload.description }}</div>
        <div class="mt-1 grid grid-cols-2 gap-1">
          <div>重量: {{ store.selectedPayload.weight }} kg</div>
          <div>附加功耗: +{{ store.selectedPayload.powerConsumption }} mAh/min</div>
        </div>
      </div>
    </div>

    <!-- Impact Assessment -->
    <div v-if="store.waypoints.length > 0 && store.currentPlan" class="border-t border-slate-700 pt-3">
      <h4 class="text-xs font-semibold text-slate-300 mb-2">📊 载荷影响评估</h4>
      <div class="grid grid-cols-3 gap-2 text-[10px]">
        <div class="bg-slate-900 rounded p-2 text-center">
          <div class="text-slate-400">预计时间</div>
          <div class="text-base font-bold text-sky-400 mt-0.5">
            {{ formatTime(store.estimatedTime) }}
          </div>
          <div
            v-if="statsDiff"
            class="text-[9px] mt-0.5"
            :class="statsDiff.timeDelta >= 0 ? 'text-red-400' : 'text-green-400'"
          >
            {{ formatDelta(statsDiff.timeDelta, 's') }}
          </div>
        </div>
        <div class="bg-slate-900 rounded p-2 text-center">
          <div class="text-slate-400">电量消耗</div>
          <div
            class="text-base font-bold mt-0.5"
            :style="{ color: batteryColor(store.batteryPercent) }"
          >
            {{ store.batteryPercent.toFixed(1) }}%
          </div>
          <div
            v-if="statsDiff"
            class="text-[9px] mt-0.5"
            :class="statsDiff.batteryDelta >= 0 ? 'text-red-400' : 'text-green-400'"
          >
            {{ formatDelta(statsDiff.batteryDelta, '%') }}
          </div>
        </div>
        <div class="bg-slate-900 rounded p-2 text-center">
          <div class="text-slate-400">安全距离</div>
          <div class="text-base font-bold text-amber-400 mt-0.5">
            {{ store.currentSafeDistance }}m
          </div>
          <div class="text-[9px] mt-0.5 text-slate-500">
            基础 {{ store.droneConfig.safeDistance }}m
          </div>
        </div>
      </div>
    </div>

    <div v-else class="text-center text-[11px] text-slate-500 py-2 bg-slate-900/50 rounded">
      选择 2 个以上航点后开始评估
    </div>
  </div>
</template>
