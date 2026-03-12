<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'

interface Props {
  title?: string
  speciesList?: string[]
  searchApiBase?: string
}

interface DensityResult {
  species: string
  representative_photo: string
  representative_painting: string
  sample_counts: {
    photo: number
    paint: number
  }
  x_axis: number[]
  photo_density: number[]
  painting_density: number[]
}

interface DynastyGroup {
  key: string
  label: string
  ages: string[]
  color: string
}

interface SegmentLayout {
  species: string
  startX: number
  width: number
  domainMin: number
  domainMax: number
}

const props = withDefaults(defineProps<Props>(), {
  title: 'Dimensions',
  speciesList: () => [],
  searchApiBase:
    typeof window !== 'undefined' ? `http://${window.location.hostname}:8001` : 'http://127.0.0.1:8001'
})

const dynastyGroups: DynastyGroup[] = [
  { key: 'tang_wudai', label: '唐/五代十国', ages: ['唐', '五代十国'], color: '#f08fa2' },
  { key: 'song', label: '宋/南宋/北宋', ages: ['宋', '南宋', '北宋'], color: '#63b68e' },
  { key: 'yuan', label: '元', ages: ['元'], color: '#59a8e2' },
  { key: 'ming', label: '明', ages: ['明'], color: '#f2a45d' },
  { key: 'qing', label: '清', ages: ['清'], color: '#ad8ef2' },
  { key: 'modern', label: '近现代', ages: ['近现代'], color: '#b27762' }
]

const selectedDynasties = ref<string[]>([])
const isLoading = ref(false)
const errorText = ref('')
const allLayerResults = ref<DensityResult[]>([])
const dynastyLayerResults = ref<Record<string, DensityResult[]>>({})
const requestSerial = ref(0)

const chartViewportRef = ref<HTMLDivElement | null>(null)
const viewportWidth = ref(980)
let resizeObserver: ResizeObserver | null = null

const axisY = 80
const chartHeight = 236
const ridgeAmplitude = 76

const sortedSpeciesList = computed(() => {
  const deduped: string[] = []
  for (const species of props.speciesList) {
    const clean = species.trim()
    if (clean && !deduped.includes(clean)) {
      deduped.push(clean)
    }
  }
  return deduped
})

const speciesOrder = computed(() => {
  if (sortedSpeciesList.value.length) {
    return sortedSpeciesList.value
  }
  return allLayerResults.value.map((item) => item.species)
})

const activeDynastyGroups = computed(() =>
  dynastyGroups.filter((group) => selectedDynasties.value.includes(group.key))
)

const displayLayers = computed(() => {
  const layers: Array<{ key: string; label: string; color: string; isAll: boolean; results: DensityResult[] }> = [
    {
      key: 'all',
      label: 'All',
      color: '#3d4f4a',
      isAll: true,
      results: allLayerResults.value
    }
  ]

  for (const group of activeDynastyGroups.value) {
    layers.push({
      key: group.key,
      label: group.label,
      color: group.color,
      isAll: false,
      results: dynastyLayerResults.value[group.key] ?? []
    })
  }

  return layers
})

const chartLayout = computed(() => {
  const layers = displayLayers.value
  const padding = 16
  const defaultGap = 10

  const findResultForSpecies = (species: string) => {
    for (const layer of layers) {
      const item = layer.results.find((res) => res.species === species)
      if (item) return item
    }
    return null
  }

  const rawItems = speciesOrder.value
    .map((species) => {
      const base = findResultForSpecies(species)
      if (!base || !base.x_axis.length) return null
      const minX = base.x_axis[0] ?? 0
      const maxX = base.x_axis[base.x_axis.length - 1] ?? 0
      const span = Math.max(0.04, maxX - minX)
      return { species, minX, maxX, span }
    })
    .filter((item): item is { species: string; minX: number; maxX: number; span: number } => Boolean(item))

  if (!rawItems.length) {
    return {
      segments: [] as SegmentLayout[],
      width: Math.max(320, viewportWidth.value)
    }
  }

  const totalSpan = rawItems.reduce((acc, item) => acc + item.span, 0)
  const segments: SegmentLayout[] = []

  const stageWidth = Math.max(320, viewportWidth.value)
  const innerWidth = Math.max(200, stageWidth - padding * 2)
  const gap = rawItems.length > 1 ? Math.min(defaultGap, innerWidth / Math.max(20, rawItems.length * 2)) : 0
  const usableWidth = Math.max(120, innerWidth - Math.max(0, rawItems.length - 1) * gap)

  let cursor = padding
  for (const item of rawItems) {
    const weight = item.span / Math.max(1e-9, totalSpan)
    const width = usableWidth * weight
    segments.push({
      species: item.species,
      startX: cursor,
      width,
      domainMin: item.minX,
      domainMax: item.maxX
    })
    cursor += width + gap
  }

  return {
    segments,
    width: stageWidth
  }
})

const densityScale = computed(() => {
  let maxPhoto = 0
  let maxPaint = 0

  for (const layer of displayLayers.value) {
    for (const item of layer.results) {
      for (const value of item.photo_density) {
        if (value > maxPhoto) maxPhoto = value
      }
      for (const value of item.painting_density) {
        if (value > maxPaint) maxPaint = value
      }
    }
  }

  return {
    photoScale: ridgeAmplitude / Math.max(1e-6, maxPhoto || 1),
    paintScale: ridgeAmplitude / Math.max(1e-6, maxPaint || 1)
  }
})

watch(
  () => [sortedSpeciesList.value.join('|'), selectedDynasties.value.join('|')],
  () => {
    void refreshData()
  },
  { immediate: true }
)

onMounted(() => {
  updateViewportWidth()
  if (chartViewportRef.value) {
    resizeObserver = new ResizeObserver(() => {
      updateViewportWidth()
    })
    resizeObserver.observe(chartViewportRef.value)
  }
})

onBeforeUnmount(() => {
  if (resizeObserver) {
    resizeObserver.disconnect()
    resizeObserver = null
  }
})

function updateViewportWidth() {
  const width = chartViewportRef.value?.clientWidth ?? 0
  if (width > 0) {
    viewportWidth.value = width
  }
}

async function refreshData() {
  if (!sortedSpeciesList.value.length) {
    allLayerResults.value = []
    dynastyLayerResults.value = {}
    errorText.value = ''
    return
  }

  const serial = requestSerial.value + 1
  requestSerial.value = serial
  isLoading.value = true
  errorText.value = ''

  try {
    const allPromise = fetchLayerResults({})
    const dynastyPromises = selectedDynasties.value.map(async (layerKey) => {
      const group = dynastyGroups.find((item) => item.key === layerKey)
      if (!group) return { key: layerKey, data: [] as DensityResult[] }
      const data = await fetchLayerResults({ age: group.ages })
      return { key: layerKey, data }
    })

    const [allData, ...dynastyData] = await Promise.all([allPromise, ...dynastyPromises])
    if (serial !== requestSerial.value) return

    allLayerResults.value = allData
    const next: Record<string, DensityResult[]> = {}
    for (const item of dynastyData) {
      next[item.key] = item.data
    }
    dynastyLayerResults.value = next
  } catch (error) {
    if (serial !== requestSerial.value) return
    allLayerResults.value = []
    dynastyLayerResults.value = {}
    errorText.value = `请求失败：${error instanceof Error ? error.message : '未知错误'}`
  } finally {
    if (serial === requestSerial.value) {
      isLoading.value = false
    }
  }
}

async function fetchLayerResults(classMethod: Record<string, unknown>) {
  const response = await fetch(`${props.searchApiBase}/search_paintings`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      species_list: sortedSpeciesList.value,
      threshold: 0.5,
      class_method: classMethod,
      structure_w: 0.8,
      color_w: 0.3,
      texture_w: 0.1
    })
  })

  if (!response.ok) {
    throw new Error(await response.text())
  }

  return (await response.json()) as DensityResult[]
}

function ridgePath(item: DensityResult, segment: SegmentLayout, type: 'photo' | 'painting') {
  const yVals = type === 'photo' ? item.photo_density : item.painting_density
  const xVals = item.x_axis
  if (!xVals.length || xVals.length !== yVals.length) {
    return ''
  }

  const scale = type === 'photo' ? densityScale.value.photoScale : densityScale.value.paintScale
  const domainSpan = Math.max(1e-6, segment.domainMax - segment.domainMin)

  const points = xVals.map((x, index) => {
    const density = yVals[index] ?? 0
    const nx = (x - segment.domainMin) / domainSpan
    const px = segment.startX + nx * segment.width
    const py = type === 'painting' ? axisY - density * scale : axisY + density * scale
    return `${px.toFixed(2)},${py.toFixed(2)}`
  })

  const startX = segment.startX.toFixed(2)
  const endX = (segment.startX + segment.width).toFixed(2)
  const baseline = axisY.toFixed(2)
  return `M ${startX} ${baseline} L ${points.join(' L ')} L ${endX} ${baseline} Z`
}

function getLayerSpeciesResult(layerKey: string, species: string) {
  if (layerKey === 'all') {
    return allLayerResults.value.find((item) => item.species === species)
  }
  return (dynastyLayerResults.value[layerKey] ?? []).find((item) => item.species === species)
}
</script>

<template>
  <section class="panel panel-bottom">
    <div class="panel-title">{{ props.title }}</div>
    <div class="dimension-grid">
      <aside class="dimension-legend">
        <div class="legend-title">History</div>

        <details class="history-dropdown">
          <summary>选择朝代叠加 (可多选)</summary>
          <div class="dropdown-list">
            <label v-for="group in dynastyGroups" :key="group.key" class="dropdown-item">
              <input v-model="selectedDynasties" type="checkbox" :value="group.key" />
              <span class="legend-dot" :style="{ background: group.color }"></span>
              <span>{{ group.label }}</span>
            </label>
          </div>
        </details>

        <div class="legend-item legend-item-all">
          <span class="legend-dot" style="background: #3d4f4a"></span>
          <span>All (默认底图)</span>
        </div>

        <div class="legend-subtitle">已选物种数: {{ sortedSpeciesList.length }}</div>
        <div class="legend-subtitle">上: painting / 下: photo (共用横轴)</div>

        <div v-if="errorText" class="legend-note legend-note-error">{{ errorText }}</div>
        <div v-else-if="!sortedSpeciesList.length" class="legend-note">请先在右侧 Gallery 选择物种</div>
      </aside>

      <div ref="chartViewportRef" class="ridgeline-wrap">
        <div class="ridgeline-stage" :style="{ width: `${chartLayout.width}px`, height: `${chartHeight}px` }">
          <div v-if="isLoading && sortedSpeciesList.length > 0" class="ridge-loading-overlay">
            <span class="loading-spinner" aria-hidden="true"></span>
            <span>正在计算山峰图...</span>
          </div>

          <div class="ridgeline-scroll" :style="{ height: `${chartHeight}px` }">
            <svg class="ridgeline-svg" :viewBox="`0 0 ${chartLayout.width} ${chartHeight}`" preserveAspectRatio="none">
              <g class="all-layer-group">
                <template v-for="segment in chartLayout.segments" :key="`all-${segment.species}`">
                  <path
                    v-if="getLayerSpeciesResult('all', segment.species)"
                    class="ridge ridge-all"
                    :d="ridgePath(getLayerSpeciesResult('all', segment.species)!, segment, 'painting')"
                  />
                  <path
                    v-if="getLayerSpeciesResult('all', segment.species)"
                    class="ridge ridge-photo-all"
                    :d="ridgePath(getLayerSpeciesResult('all', segment.species)!, segment, 'photo')"
                  />
                </template>
              </g>

              <template v-for="layer in displayLayers.filter((item) => !item.isAll)" :key="layer.key">
                <g>
                  <template v-for="segment in chartLayout.segments" :key="`${layer.key}-${segment.species}`">
                    <path
                      v-if="getLayerSpeciesResult(layer.key, segment.species)"
                      class="ridge"
                      :style="{ fill: layer.color }"
                      :d="ridgePath(getLayerSpeciesResult(layer.key, segment.species)!, segment, 'painting')"
                    />
                  </template>
                </g>
              </template>

              <line class="baseline" :x1="0" :x2="chartLayout.width" :y1="axisY" :y2="axisY" />
            </svg>

            <div class="species-labels">
              <span
                v-for="segment in chartLayout.segments"
                :key="`label-${segment.species}`"
                class="species-label"
                :style="{ left: `${segment.startX + segment.width / 2}px` }"
              >
                {{ segment.species }}
              </span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </section>
</template>

<style scoped>
.panel {
  background: var(--panel-bg);
  border-radius: 18px;
  border: 1px solid var(--panel-border);
  padding: 14px;
  box-shadow: var(--shadow-soft);
  min-height: 0;
  height: 100%;
}

.panel-title {
  font-weight: 700;
  font-size: 14px;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #5f6b62;
  margin-bottom: 10px;
}

.panel-bottom {
  overflow: hidden;
}

.dimension-grid {
  display: grid;
  grid-template-columns: 220px minmax(0, 1fr);
  gap: 12px;
  align-items: stretch;
  min-height: 0;
  height: 100%;
}

.dimension-legend {
  display: flex;
  flex-direction: column;
  gap: 8px;
  font-size: 12px;
  color: #57635d;
}

.legend-title {
  font-weight: 700;
  color: #4d5852;
}

.history-dropdown {
  border: 1px solid #d8dfd6;
  border-radius: 10px;
  background: #ffffff;
  overflow: hidden;
}

.history-dropdown > summary {
  list-style: none;
  cursor: pointer;
  padding: 7px 9px;
  font-weight: 600;
  color: #3d4f4a;
}

.history-dropdown > summary::-webkit-details-marker {
  display: none;
}

.dropdown-list {
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 0 9px 9px;
}

.dropdown-item,
.legend-item {
  display: flex;
  align-items: center;
  gap: 7px;
}

.legend-item-all {
  opacity: 0.72;
}

.legend-dot {
  width: 14px;
  height: 7px;
  border-radius: 999px;
  display: inline-block;
}

.legend-subtitle {
  color: #6c746f;
}

.legend-note {
  color: #5f6d66;
}

.legend-note-error {
  color: #b94a48;
}

.ridgeline-wrap {
  min-width: 0;
  min-height: 0;
  height: 100%;
  margin-top: -10px;
}

.ridgeline-stage {
  position: relative;
  height: 100%;
}

.ridgeline-scroll {
  width: 100%;
  overflow: hidden;
  height: 100%;
  border: none;
  border-radius: 0;
  background: transparent;
}

.ridgeline-svg {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
}

.all-layer-group {
  isolation: isolate;
}

.ridge {
  opacity: 0.38;
}

.ridge-all {
  fill: #3d4f4a;
  opacity: 0.11;
}

.ridge-photo-all {
  fill: #3d4f4a;
  opacity: 0.2;
}

.baseline {
  stroke: rgba(74, 90, 83, 0.35);
  stroke-width: 1;
  stroke-dasharray: 2 4;
}

.species-labels {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 18px;
  height: 20px;
  pointer-events: none;
}

.ridge-loading-overlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  gap: 10px;
  color: #5b6b64;
  font-size: 13px;
  background: rgba(246, 249, 245, 0.55);
  z-index: 5;
}

.loading-spinner {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  border: 2px solid rgba(90, 108, 100, 0.24);
  border-top-color: #4f6b62;
  animation: ridge-spin 0.75s linear infinite;
}

@keyframes ridge-spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.species-label {
  position: absolute;
  transform: translateX(-50%);
  font-size: 11px;
  color: #607068;
  white-space: nowrap;
}

@media (max-width: 1200px) {
  .dimension-grid {
    grid-template-columns: 1fr;
  }

  .ridgeline-scroll {
    height: 204px !important;
  }
}
</style>
