<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { toDisplaySpeciesName } from '../constants/speciesMapping'

interface Props {
  title?: string
  speciesList?: string[]
  searchApiBase?: string
  layoutMode?: 'overlay' | 'stacked'
  dataset?: 'bird' | 'flower'
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

interface PaintingMeta {
  id?: string
  title?: string
  original_title?: string
  author?: string
  age?: string
  image_path?: string
}

interface HoverPreviewState {
  type: 'painting' | 'photo'
  imageUrl: string
  species: string
  meta: PaintingMeta | null
}

interface RepresentativeBadge {
  species: string
  paintingPath: string
  photoPath: string
  paintingUrl: string
  photoUrl: string
  paintingX: number
  paintingY: number
  photoX: number
  photoY: number
}

const props = withDefaults(defineProps<Props>(), {
  title: 'Dimensions',
  speciesList: () => [],
  layoutMode: 'overlay',
  dataset: 'bird',
  searchApiBase:
    typeof window !== 'undefined' ? `http://${window.location.hostname}:8001` : 'http://127.0.0.1:8001'
})

const Chinese_Mode = true

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
const paintingMetaMap = ref<Record<string, PaintingMeta>>({})
const hoveredPreview = ref<HoverPreviewState | null>(null)
const hoverPreviewPos = ref({ x: 16, y: 16 })

const chartViewportRef = ref<HTMLDivElement | null>(null)
const viewportWidth = ref(980)
const viewportHeight = ref(260)
let resizeObserver: ResizeObserver | null = null

const chartHeight = computed(() => clamp(Math.max(220, viewportHeight.value - 4), 220, 440))
const axisY = computed(() => chartHeight.value * 0.5)
const ridgeAmplitude = computed(() => Math.max(74, chartHeight.value * 0.42))

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

  const rawItems = speciesOrder.value
    .map((species) => {
      const speciesResults = layers
        .flatMap((layer) => layer.results)
        .filter((item) => item.species === species && item.x_axis.length > 0)

      if (!speciesResults.length) return null

      // 每个物种使用“所有层(all+朝代)”的并集横轴范围，确保上下山峰和叠加层严格落在同一段内。
      let minX = Number.POSITIVE_INFINITY
      let maxX = Number.NEGATIVE_INFINITY
      for (const item of speciesResults) {
        const left = item.x_axis[0] ?? 0
        const right = item.x_axis[item.x_axis.length - 1] ?? 0
        if (left < minX) minX = left
        if (right > maxX) maxX = right
      }

      if (!Number.isFinite(minX) || !Number.isFinite(maxX)) return null
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
    photoScale: ridgeAmplitude.value / Math.max(1e-6, maxPhoto || 1),
    paintScale: ridgeAmplitude.value / Math.max(1e-6, maxPaint || 1)
  }
})

const dynastyOverlayScale = computed(() => {
  if (!activeDynastyGroups.value.length) {
    return 1
  }
  return 2
})

const representativeBadges = computed<RepresentativeBadge[]>(() => {
  const badges: RepresentativeBadge[] = []

  for (const segment of chartLayout.value.segments) {
    const result = getLayerSpeciesResult('all', segment.species)
    if (!result) continue

    const paintingPeak = getPeakAnchor(result, segment, 'painting')
    const photoPeak = getPeakAnchor(result, segment, 'photo')

    badges.push({
      species: segment.species,
      paintingPath: result.representative_painting,
      photoPath: result.representative_photo,
      paintingUrl: resolveImageUrl(result.representative_painting),
      photoUrl: resolveImageUrl(result.representative_photo),
      paintingX: paintingPeak.x,
      paintingY: clamp(paintingPeak.y - 22, 16, axisY.value - 12),
      photoX: photoPeak.x,
      photoY: clamp(photoPeak.y + 22, axisY.value + 12, chartHeight.value - 16)
    })
  }

  return badges
})

watch(
  () => [sortedSpeciesList.value.join('|'), selectedDynasties.value.join('|'), props.dataset],
  () => {
    void refreshData()
  },
  { immediate: true }
)

watch(
  () => props.dataset,
  () => {
    void loadPaintingMeta()
  }
)

onMounted(() => {
  void loadPaintingMeta()
  updateViewportWidth()
  if (chartViewportRef.value) {
    resizeObserver = new ResizeObserver(() => {
      updateViewportWidth()
    })
    resizeObserver.observe(chartViewportRef.value)
  }
})

onBeforeUnmount(() => {
  hoveredPreview.value = null
  if (resizeObserver) {
    resizeObserver.disconnect()
    resizeObserver = null
  }
})

async function loadPaintingMeta() {
  const metadataFile = props.dataset === 'flower' ? 'flower_painting_metadata.json' : 'bird_painting_metadata.json'
  try {
    const response = await fetch(`${props.searchApiBase}/data/${metadataFile}`)
    if (!response.ok) {
      return
    }
    const data = (await response.json()) as PaintingMeta[]
    const nextMap: Record<string, PaintingMeta> = {}
    for (const item of data) {
      const imagePath = String(item.image_path ?? '').trim()
      if (!imagePath) continue
      const key = normalizePaintingMetaKey(imagePath)
      if (!key) continue
      nextMap[key] = item
    }
    paintingMetaMap.value = nextMap
  } catch {
    paintingMetaMap.value = {}
  }
}

function updateViewportWidth() {
  const width = chartViewportRef.value?.clientWidth ?? 0
  const height = chartViewportRef.value?.clientHeight ?? 0
  if (width > 0) {
    viewportWidth.value = width
  }
  if (height > 0) {
    viewportHeight.value = height
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
      dataset: props.dataset,
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

function ridgePath(
  item: DensityResult,
  segment: SegmentLayout,
  type: 'photo' | 'painting',
  extraScale = 1
) {
  const yVals = type === 'photo' ? item.photo_density : item.painting_density
  const xVals = item.x_axis
  if (!xVals.length || xVals.length !== yVals.length) {
    return ''
  }

  const scale = type === 'photo' ? densityScale.value.photoScale : densityScale.value.paintScale
  const domainSpan = Math.max(1e-6, segment.domainMax - segment.domainMin)

  const points = xVals.map((x, index) => {
    const density = yVals[index] ?? 0
    const nx = clamp((x - segment.domainMin) / domainSpan, 0, 1)
    const px = segment.startX + nx * segment.width
    const py =
      type === 'painting'
        ? axisY.value - density * scale * extraScale
        : axisY.value + density * scale
    return `${px.toFixed(2)},${py.toFixed(2)}`
  })

  const startX = segment.startX.toFixed(2)
  const endX = (segment.startX + segment.width).toFixed(2)
  const baseline = axisY.value.toFixed(2)
  return `M ${startX} ${baseline} L ${points.join(' L ')} L ${endX} ${baseline} Z`
}

function sampleDensityAtX(item: DensityResult, type: 'photo' | 'painting', x: number) {
  const xVals = item.x_axis
  const yVals = type === 'photo' ? item.photo_density : item.painting_density
  if (!xVals.length || xVals.length !== yVals.length) return 0

  const firstX = xVals[0] ?? 0
  const lastX = xVals[xVals.length - 1] ?? 0
  if (x < firstX || x > lastX) return 0

  let lo = 0
  let hi = xVals.length - 1
  while (lo + 1 < hi) {
    const mid = (lo + hi) >> 1
    const mv = xVals[mid] ?? 0
    if (mv <= x) lo = mid
    else hi = mid
  }

  const x0 = xVals[lo] ?? x
  const x1 = xVals[hi] ?? x
  const y0 = yVals[lo] ?? 0
  const y1 = yVals[hi] ?? y0
  if (Math.abs(x1 - x0) < 1e-9) return y0
  const t = (x - x0) / (x1 - x0)
  return y0 + (y1 - y0) * t
}

function stackedRidgePath(segment: SegmentLayout, layerKey: string) {
  const layerIdx = activeDynastyGroups.value.findIndex((group) => group.key === layerKey)
  if (layerIdx < 0) return ''

  const current = getLayerSpeciesResult(layerKey, segment.species)
  if (!current) return ''

  const sampleCount = 201
  const pointsTop: string[] = []
  const pointsBottom: string[] = []
  const span = Math.max(1e-6, segment.domainMax - segment.domainMin)

  for (let i = 0; i < sampleCount; i += 1) {
    const t = i / Math.max(1, sampleCount - 1)
    const xDomain = segment.domainMin + span * t
    const px = segment.startX + segment.width * t

    let cumDensity = 0
    for (let j = 0; j < layerIdx; j += 1) {
      const prevKey = activeDynastyGroups.value[j]?.key
      if (!prevKey) continue
      const prev = getLayerSpeciesResult(prevKey, segment.species)
      if (!prev) continue
      cumDensity += sampleDensityAtX(prev, 'painting', xDomain)
    }

    const currDensity = sampleDensityAtX(current, 'painting', xDomain)
    const yBottom = axisY.value - cumDensity * densityScale.value.paintScale
    const yTop = axisY.value - (cumDensity + currDensity) * densityScale.value.paintScale

    pointsTop.push(`${px.toFixed(2)},${yTop.toFixed(2)}`)
    pointsBottom.push(`${px.toFixed(2)},${yBottom.toFixed(2)}`)
  }

  if (!pointsTop.length) return ''
  return `M ${pointsBottom[0]} L ${pointsTop.join(' L ')} L ${pointsBottom.reverse().join(' L ')} Z`
}

function getPeakAnchor(item: DensityResult, segment: SegmentLayout, type: 'photo' | 'painting') {
  const yVals = type === 'photo' ? item.photo_density : item.painting_density
  const xVals = item.x_axis
  if (!xVals.length || xVals.length !== yVals.length) {
    return {
      x: segment.startX + segment.width / 2,
      y: axisY.value
    }
  }

  let maxIndex = 0
  let maxValue = Number.NEGATIVE_INFINITY
  for (let i = 0; i < yVals.length; i += 1) {
    const value = yVals[i] ?? 0
    if (value > maxValue) {
      maxValue = value
      maxIndex = i
    }
  }

  const x = xVals[maxIndex] ?? xVals[0] ?? 0
  const density = yVals[maxIndex] ?? 0
  const scale = type === 'photo' ? densityScale.value.photoScale : densityScale.value.paintScale
  const domainSpan = Math.max(1e-6, segment.domainMax - segment.domainMin)
  const nx = clamp((x - segment.domainMin) / domainSpan, 0, 1)
  const px = segment.startX + nx * segment.width
  const py = type === 'painting' ? axisY.value - density * scale : axisY.value + density * scale

  return {
    x: px,
    y: py
  }
}

function resolveImageUrl(path: string) {
  if (!path) return ''
  if (/^https?:\/\//i.test(path)) return path

  const base = props.searchApiBase.replace(/\/$/, '')
  let normalized = path.replace(/\\/g, '/')

  const dataIdx = normalized.indexOf('/data/')
  if (dataIdx >= 0) {
    normalized = normalized.slice(dataIdx)
  } else if (normalized.startsWith('./data/')) {
    normalized = normalized.slice(1)
  } else if (normalized.startsWith('data/')) {
    normalized = `/${normalized}`
  } else if (normalized.startsWith('./')) {
    normalized = normalized.slice(1)
  } else if (!normalized.startsWith('/')) {
    normalized = `/${normalized}`
  }

  const encodedPath = normalized
    .split('/')
    .map((segment, idx) => (idx === 0 ? segment : encodeURIComponent(segment)))
    .join('/')

  return `${base}${encodedPath}`
}

function normalizePaintingMetaKey(path: string) {
  let normalized = path.replace(/\\/g, '/').trim()
  if (!normalized) return ''

  const downloadedFlowerIdx = normalized.indexOf('/downloaded_paintings-flower/')
  if (downloadedFlowerIdx >= 0) {
    normalized = normalized.slice(downloadedFlowerIdx + '/downloaded_paintings-flower/'.length)
  }

  const downloadedIdx = normalized.indexOf('/downloaded_paintings/')
  if (downloadedIdx >= 0) {
    normalized = normalized.slice(downloadedIdx + '/downloaded_paintings/'.length)
  }

  if (normalized.startsWith('./')) {
    normalized = normalized.slice(2)
  }
  if (normalized.startsWith('/')) {
    normalized = normalized.slice(1)
  }
  if (normalized.startsWith('data/')) {
    normalized = normalized.slice('data/'.length)
  }
  if (normalized.startsWith('downloaded_paintings/')) {
    normalized = normalized.slice('downloaded_paintings/'.length)
  }
  if (normalized.startsWith('downloaded_paintings-flower/')) {
    normalized = normalized.slice('downloaded_paintings-flower/'.length)
  }
  return normalized
}

function findPaintingMeta(path: string): PaintingMeta | null {
  const key = normalizePaintingMetaKey(path)
  if (!key) return null
  return paintingMetaMap.value[key] ?? null
}

function updateHoverPreviewPosition(event: MouseEvent) {
  const cardWidth = 250
  const cardHeight = 240
  const offset = 24
  const vw = typeof window !== 'undefined' ? window.innerWidth : 1200
  const vh = typeof window !== 'undefined' ? window.innerHeight : 800

  // 使用视口级 fixed 定位，避免被图表/面板的 overflow 裁切。
  let x = event.clientX - cardWidth - offset
  let y = event.clientY - cardHeight - offset

  x = clamp(x, 8, Math.max(8, vw - cardWidth - 8))
  y = clamp(y, 8, Math.max(8, vh - cardHeight - 8))
  hoverPreviewPos.value = { x, y }
}

function onBadgeEnter(badge: RepresentativeBadge, type: 'painting' | 'photo', event: MouseEvent) {
  updateHoverPreviewPosition(event)
  if (type === 'painting') {
    hoveredPreview.value = {
      type,
      imageUrl: badge.paintingUrl,
      species: badge.species,
      meta: findPaintingMeta(badge.paintingPath)
    }
    return
  }

  hoveredPreview.value = {
    type,
    imageUrl: badge.photoUrl,
    species: badge.species,
    meta: null
  }
}

function onBadgeMove(event: MouseEvent) {
  if (!hoveredPreview.value) return
  updateHoverPreviewPosition(event)
}

function onBadgeLeave() {
  hoveredPreview.value = null
}

function clamp(value: number, min: number, max: number) {
  return Math.max(min, Math.min(max, value))
}

function displaySpeciesName(species: string) {
  return toDisplaySpeciesName(species, Chinese_Mode)
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

              <template v-if="props.layoutMode === 'overlay'">
                <template v-for="layer in displayLayers.filter((item) => !item.isAll)" :key="layer.key">
                  <g>
                    <template v-for="segment in chartLayout.segments" :key="`${layer.key}-${segment.species}`">
                      <path
                        v-if="getLayerSpeciesResult(layer.key, segment.species)"
                        class="ridge"
                        :style="{ fill: layer.color }"
                        :d="ridgePath(getLayerSpeciesResult(layer.key, segment.species)!, segment, 'painting', dynastyOverlayScale)"
                      />
                    </template>
                  </g>
                </template>
              </template>

              <template v-else>
                <template v-for="group in activeDynastyGroups" :key="group.key">
                  <g>
                    <template v-for="segment in chartLayout.segments" :key="`stack-${group.key}-${segment.species}`">
                      <path
                        v-if="getLayerSpeciesResult(group.key, segment.species)"
                        class="ridge"
                        :style="{ fill: group.color }"
                        :d="stackedRidgePath(segment, group.key)"
                      />
                    </template>
                  </g>
                </template>
              </template>

              <line class="baseline" :x1="0" :x2="chartLayout.width" :y1="axisY" :y2="axisY" />
            </svg>

            <div class="representative-layer">
              <template v-for="badge in representativeBadges" :key="`badge-${badge.species}`">
                <div
                  v-if="badge.paintingUrl"
                  class="representative-badge representative-badge-painting"
                  :style="{ left: `${badge.paintingX}px`, top: `${badge.paintingY}px` }"
                  :title="`${displaySpeciesName(badge.species)} painting`"
                  @mouseenter="onBadgeEnter(badge, 'painting', $event)"
                  @mousemove="onBadgeMove($event)"
                  @mouseleave="onBadgeLeave"
                >
                  <img :src="badge.paintingUrl" :alt="`${displaySpeciesName(badge.species)} painting`" loading="lazy" decoding="async" />
                </div>

                <div
                  v-if="badge.photoUrl"
                  class="representative-badge representative-badge-photo"
                  :style="{ left: `${badge.photoX}px`, top: `${badge.photoY}px` }"
                  :title="`${displaySpeciesName(badge.species)} photo`"
                  @mouseenter="onBadgeEnter(badge, 'photo', $event)"
                  @mousemove="onBadgeMove($event)"
                  @mouseleave="onBadgeLeave"
                >
                  <img :src="badge.photoUrl" :alt="`${displaySpeciesName(badge.species)} photo`" loading="lazy" decoding="async" />
                </div>
              </template>
            </div>

            <div class="species-labels">
              <span
                v-for="segment in chartLayout.segments"
                :key="`label-${segment.species}`"
                class="species-label"
                :style="{ left: `${segment.startX + segment.width / 2}px` }"
              >
                {{ displaySpeciesName(segment.species) }}
              </span>
            </div>
          </div>

          <div
            v-if="hoveredPreview"
            class="hover-preview-card"
            :style="{ left: `${hoverPreviewPos.x}px`, top: `${hoverPreviewPos.y}px` }"
          >
            <div class="hover-preview-image-wrap">
              <img :src="hoveredPreview.imageUrl" :alt="displaySpeciesName(hoveredPreview.species)" class="hover-preview-image" />
            </div>
            <div class="hover-preview-meta">
              <div class="hover-preview-species">{{ displaySpeciesName(hoveredPreview.species) }}</div>
              <template v-if="hoveredPreview.type === 'painting' && hoveredPreview.meta">
                <div class="hover-preview-line">画作：{{ hoveredPreview.meta.title || hoveredPreview.meta.original_title || '未知' }}</div>
                <div class="hover-preview-line">朝代：{{ hoveredPreview.meta.age || '未知' }}</div>
                <div class="hover-preview-line">作者：{{ hoveredPreview.meta.author || '未知' }}</div>
              </template>
              <template v-else-if="hoveredPreview.type === 'painting'">
                <div class="hover-preview-line">画作元信息缺失</div>
              </template>
              <template v-else>
                <div class="hover-preview-line">类型：摄影参考图</div>
              </template>
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
  bottom: 12px;
  height: 20px;
  pointer-events: none;
}

.representative-layer {
  position: absolute;
  inset: 0;
  pointer-events: none;
}

.representative-badge {
  position: absolute;
  width: 34px;
  height: 34px;
  transform: translate(-50%, -50%);
  border-radius: 50%;
  overflow: hidden;
  border: 1px solid rgba(255, 255, 255, 0.92);
  box-shadow: 0 2px 6px rgba(24, 32, 28, 0.28);
  background: #ffffff;
  pointer-events: auto;
  cursor: pointer;
}

.representative-badge img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.representative-badge-painting {
  outline: 1px solid rgba(215, 123, 136, 0.55);
}

.representative-badge-photo {
  outline: 1px solid rgba(97, 126, 185, 0.5);
}

.hover-preview-card {
  position: fixed;
  width: 250px;
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.95);
  border: 1px solid rgba(216, 223, 214, 0.9);
  box-shadow: 0 10px 24px rgba(29, 40, 36, 0.2);
  padding: 8px;
  z-index: 1200;
  backdrop-filter: blur(4px);
  pointer-events: none;
}

.hover-preview-image-wrap {
  width: 100%;
  height: 148px;
  border-radius: 8px;
  overflow: hidden;
  background: #f3f6f2;
}

.hover-preview-image {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.hover-preview-meta {
  margin-top: 8px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.hover-preview-species {
  font-size: 12px;
  font-weight: 700;
  color: #3f5048;
}

.hover-preview-line {
  font-size: 12px;
  color: #5f6d66;
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
