<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import uploadIconUrl from '../assets/上传图标.svg'

type WeightKey = 'structure' | 'color' | 'texture'

interface WeightItem {
  key: WeightKey
  label: string
  value: number
  tone?: 'pink' | 'mint' | 'blue'
}

interface SearchItem {
  path: string
  is_representative: boolean
  label: string
  score: number
  radius: number
  angle: number
}

interface OrbitPoint {
  x: number
  y: number
  raw: SearchItem
}

const SHADOW_POINTS_PER_SPECIES = 10

interface RepresentativeNode {
  x: number
  y: number
  raw: SearchItem
  imageUrl: string
}

interface GalleryItem {
  id: string
  imageUrl: string
  species: string
}

interface Props {
  title?: string
  promptPlaceholder?: string
  weights?: WeightItem[]
  referenceImage?: string
  galleryImages?: string[]
  searchApiBase?: string
  promptApiBase?: string
  controlsTarget?: string
  searchFile?: File | null
}

const props = withDefaults(defineProps<Props>(), {
  title: 'Compare',
  promptPlaceholder: 'Describe the style, palette, and mood',
  weights: () => [
    { key: 'structure', label: 'Structure', value: 1, tone: 'mint' },
    { key: 'color', label: 'Color', value: 0.3, tone: 'blue' },
    { key: 'texture', label: 'Texture', value: 0.2, tone: 'pink' }
  ],
  referenceImage: '',
  galleryImages: () => [],
  searchApiBase:
    typeof window !== 'undefined' ? `http://${window.location.hostname}:8001` : 'http://127.0.0.1:8001',
  promptApiBase:
    typeof window !== 'undefined' ? `http://${window.location.hostname}:8000` : 'http://127.0.0.1:8000',
  controlsTarget: '',
  searchFile: null
})

const emit = defineEmits<{
  (e: 'species-change', speciesList: string[]): void
}>()

const weightRows = ref<WeightItem[]>(
  props.weights.map((weight) => ({ ...weight, value: clamp(weight.value, 0, 1) }))
)
const searchResults = ref<SearchItem[]>([])
const isSearching = ref(false)
const isPromptLoading = ref(false)
const searchError = ref('')
const promptError = ref('')
const promptText = ref('')
const hoverItem = ref<SearchItem | null>(null)
const pinnedItem = ref<SearchItem | null>(null)
const galleryUploads = ref<GalleryItem[]>([])
const selectedSpecies = ref<string[]>([])
const needsConfirmSpecies = ref(false)
const orbitRef = ref<HTMLDivElement | null>(null)
const orbitScale = ref(1)
const orbitOffsetX = ref(0)
const orbitOffsetY = ref(0)
const isPanning = ref(false)
const shadowRadius = ref(4)
const needsConfirmSearch = ref(false)
const imageScale = ref(1)
const imageOffsetX = ref(0)
const imageOffsetY = ref(0)
const isImageDragging = ref(false)

let panStartX = 0
let panStartY = 0
let panOriginX = 0
let panOriginY = 0
let imageDragStartX = 0
let imageDragStartY = 0
let imageDragOriginX = 0
let imageDragOriginY = 0

let debounceTimer: ReturnType<typeof setTimeout> | null = null
let activeController: AbortController | null = null
let orbitResizeObserver: ResizeObserver | null = null
let controlsHostProbeTimer: ReturnType<typeof setInterval> | null = null

const orbitRadiusStats = computed(() => {
  const radii = searchResults.value.map((item) => Math.max(0, item.radius))
  if (!radii.length) {
    return { min: 0, max: 1 }
  }

  let min = Number.POSITIVE_INFINITY
  let max = Number.NEGATIVE_INFINITY
  for (const r of radii) {
    if (r < min) min = r
    if (r > max) max = r
  }

  if (!Number.isFinite(min) || !Number.isFinite(max)) {
    return { min: 0, max: 1 }
  }

  if (max - min < 1e-4) {
    return { min, max: min + 1 }
  }

  return { min, max }
})

function updateShadowRadius() {
  if (!orbitRef.value) return
  const size = Math.min(orbitRef.value.clientWidth, orbitRef.value.clientHeight)
  if (size <= 0) return

  // 阴影点略小于代表图，保证可见性且层级清晰。
  const r = (17 * 100) / size
  shadowRadius.value = clamp(r, 2.1, 7.6)
}

function spreadRepresentativeNodes(nodes: RepresentativeNode[]) {
  if (nodes.length < 2) return nodes

  // 多轮排斥 + 回弹，尽量避免重叠同时保留目标环位置信息。
  const collisionDist = 7.4
  const iterations = 20
  const maxShift = 13

  const original = nodes.map((node) => ({ x: node.x, y: node.y }))
  const pos = nodes.map((node) => ({ x: node.x, y: node.y }))

  for (let iter = 0; iter < iterations; iter += 1) {
    for (let i = 0; i < pos.length; i += 1) {
      const pi = pos[i]!
      for (let j = i + 1; j < pos.length; j += 1) {
        const pj = pos[j]!
        const dx = pj.x - pi.x
        const dy = pj.y - pi.y
        const d2 = dx * dx + dy * dy

        if (d2 >= collisionDist * collisionDist) continue

        let dist = Math.sqrt(d2)
        if (dist < 1e-6) {
          const jitter = ((i + 1) * (j + 3)) % 7
          dist = 0.001
          pj.x += (jitter - 3) * 0.12
          pj.y += (3 - jitter) * 0.12
        }

        // 推开量只覆盖“重叠部分”，并故意保守，避免排成均匀圆环。
        const overlap = collisionDist - dist
        const softness = 0.42 + 0.4 * (iter / Math.max(1, iterations - 1))
        const pushX = (dx / dist) * overlap * softness * 0.5
        const pushY = (dy / dist) * overlap * softness * 0.5

        pi.x -= pushX
        pi.y -= pushY
        pj.x += pushX
        pj.y += pushY
      }
    }

    // 轻微回弹，保持整体布局语义不被过度破坏。
    for (let i = 0; i < pos.length; i += 1) {
      const pi = pos[i]!
      const oi = original[i]!
      // 回弹：贴近目标位置，但保留足够排斥后的分散性。
      pi.x += (oi.x - pi.x) * 0.19
      pi.y += (oi.y - pi.y) * 0.19

      const offX = pi.x - oi.x
      const offY = pi.y - oi.y
      const offLen = Math.sqrt(offX * offX + offY * offY)
      if (offLen > maxShift) {
        const k = maxShift / offLen
        pi.x = oi.x + offX * k
        pi.y = oi.y + offY * k
      }

      const clamped = clampToOrbitCircle(pi.x, pi.y, 48.2)
      pi.x = clamped.x
      pi.y = clamped.y
    }
  }

  return nodes.map((node, idx) => ({
    ...node,
    x: pos[idx]!.x,
    y: pos[idx]!.y
  }))
}

const representativeNodes = computed<RepresentativeNode[]>(() =>
  spreadRepresentativeNodes(
    buildTieredRepresentativeTargets(searchResults.value.filter((item) => item.is_representative))
  )
)

function buildTieredRepresentativeTargets(items: SearchItem[]): RepresentativeNode[] {
  if (!items.length) return []

  const enriched = items
    .map((item) => {
      const base = toOrbitPoint(item)
      const angle = ((Math.atan2(base.y - 50, base.x - 50) * 180) / Math.PI + 360) % 360
      const radius = Math.hypot(base.x - 50, base.y - 50)
      return {
        item,
        baseAngle: angle,
        baseRadius: radius,
        imageUrl: resolveImageUrl(item.path)
      }
    })
    .sort((a, b) => a.baseRadius - b.baseRadius)

  const total = enriched.length
  const innerCount = total <= 6 ? 1 : Math.max(1, Math.round(total * 0.12))
  const middleCount = total <= 8 ? 2 : Math.max(2, Math.round(total * 0.2))
  const outerCount = Math.max(0, total - innerCount - middleCount)

  const inner = enriched.slice(0, innerCount)
  const middle = enriched.slice(innerCount, innerCount + middleCount)
  const outer = enriched.slice(innerCount + middleCount, innerCount + middleCount + outerCount)

  const nodes: RepresentativeNode[] = []
  nodes.push(...layoutRing(inner, [10, 16]))
  nodes.push(...layoutRing(middle, [26, 33]))
  nodes.push(...layoutRing(outer, [37, 48]))
  return nodes
}

function layoutRing(
  ringItems: Array<{ item: SearchItem; baseAngle: number; imageUrl: string }>,
  radiusRange: [number, number]
): RepresentativeNode[] {
  if (!ringItems.length) return []

  const sorted = [...ringItems].sort((a, b) => a.baseAngle - b.baseAngle)
  const step = 360 / Math.max(1, sorted.length)

  return sorted.map((entry, index) => {
    const regularAngle = index * step
    const seed = hashSeed(entry.item.path)
    const jitterAngle = ((seed % 1000) / 1000 - 0.5) * 10
    const angle = blendAngles(entry.baseAngle, regularAngle, 0.44) + jitterAngle

    const radiusBase = radiusRange[0] + (radiusRange[1] - radiusRange[0]) * ((index + 0.5) / sorted.length)
    const radiusJitter = (((seed >>> 4) % 1000) / 1000 - 0.5) * 2
    const radius = clamp(radiusBase + radiusJitter, radiusRange[0], radiusRange[1])
    const radian = (angle * Math.PI) / 180
    const point = clampToOrbitCircle(50 + Math.cos(radian) * radius, 50 + Math.sin(radian) * radius, 47.6)

    return {
      x: point.x,
      y: point.y,
      raw: entry.item,
      imageUrl: entry.imageUrl
    }
  })
}

function blendAngles(a: number, b: number, blend: number) {
  const aRad = (a * Math.PI) / 180
  const bRad = (b * Math.PI) / 180
  const x = (1 - blend) * Math.cos(aRad) + blend * Math.cos(bRad)
  const y = (1 - blend) * Math.sin(aRad) + blend * Math.sin(bRad)
  return ((Math.atan2(y, x) * 180) / Math.PI + 360) % 360
}

const representativeAnchorBySpecies = computed(() => {
  const map = new Map<string, { x: number; y: number }>()
  for (const node of representativeNodes.value) {
    const species = node.raw.label.trim()
    if (!species || map.has(species)) continue
    map.set(species, { x: node.x, y: node.y })
  }
  return map
})

const shadowPoints = computed(() => {
  return searchResults.value
    .filter((item) => !item.is_representative)
    .map((item) => {
      const base = toOrbitPoint(item)
      const species = item.label.trim()
      const anchor = representativeAnchorBySpecies.value.get(species)

      if (!anchor) {
        return base
      }

      const pulled = pullShadowPointTowardAnchor(base, anchor, item.path)
      return {
        ...pulled,
        raw: item
      }
    })
})

const activePreviewItem = computed(() => pinnedItem.value ?? hoverItem.value)

const highlightedSpecies = computed(() => (pinnedItem.value ?? hoverItem.value)?.label ?? '')

const orbitCenterImageUrl = computed(() => props.referenceImage || '')

const displayImageUrl = computed(() => {
  if (activePreviewItem.value) {
    return resolveImageUrl(activePreviewItem.value.path)
  }
  if (props.referenceImage) return props.referenceImage
  const first = searchResults.value[0]
  if (!first) return ''
  return resolveImageUrl(first.path)
})

const galleryResultImages = computed<GalleryItem[]>(() => galleryUploads.value)

const arcStroke = computed(() => ({
  structure: String(getWeightValue('structure')),
  color: String(getWeightValue('color')),
  texture: String(getWeightValue('texture'))
}))

const orbitSceneStyle = computed(() => ({
  transform: `translate(${orbitOffsetX.value}px, ${orbitOffsetY.value}px) scale(${orbitScale.value})`
}))

const previewImageStyle = computed(() => ({
  transform: `translate(${imageOffsetX.value}px, ${imageOffsetY.value}px) scale(${imageScale.value})`
}))

const controlsHostEl = ref<HTMLElement | null>(null)

function syncControlsHostReady() {
  const target = props.controlsTarget?.trim()
  if (!target || typeof document === 'undefined') {
    controlsHostEl.value = null
    return
  }
  controlsHostEl.value = document.querySelector(target) as HTMLElement | null
}

const useExternalControls = computed(() =>
  Boolean(props.controlsTarget && props.controlsTarget.trim() && controlsHostEl.value)
)

watch(
  () => props.searchFile,
  (file) => {
    if (!file) {
      searchResults.value = []
      searchError.value = ''
      hoverItem.value = null
      pinnedItem.value = null
      needsConfirmSearch.value = false
      resetPreviewTransform()
      return
    }
    searchResults.value = []
    searchError.value = ''
    hoverItem.value = null
    pinnedItem.value = null
    needsConfirmSearch.value = true
    resetPreviewTransform()
  },
  { immediate: true }
)

onBeforeUnmount(() => {
  if (debounceTimer) {
    clearTimeout(debounceTimer)
  }
  if (activeController) {
    activeController.abort()
  }
  if (orbitResizeObserver) {
    orbitResizeObserver.disconnect()
    orbitResizeObserver = null
  }
  if (controlsHostProbeTimer) {
    clearInterval(controlsHostProbeTimer)
    controlsHostProbeTimer = null
  }
  galleryUploads.value.forEach((item) => URL.revokeObjectURL(item.imageUrl))
})

onMounted(() => {
  syncControlsHostReady()
  // 某些情况下 Teleport 目标会晚于当前组件挂载，轮询短暂探测确保命中。
  requestAnimationFrame(() => {
    syncControlsHostReady()
  })
  controlsHostProbeTimer = setInterval(() => {
    syncControlsHostReady()
    if (controlsHostEl.value && controlsHostProbeTimer) {
      clearInterval(controlsHostProbeTimer)
      controlsHostProbeTimer = null
    }
  }, 250)

  updateShadowRadius()
  if (orbitRef.value) {
    orbitResizeObserver = new ResizeObserver(() => {
      updateShadowRadius()
    })
    orbitResizeObserver.observe(orbitRef.value)
  }
})

watch(
  () => props.controlsTarget,
  () => {
    syncControlsHostReady()
    if (!controlsHostEl.value && !controlsHostProbeTimer) {
      controlsHostProbeTimer = setInterval(() => {
        syncControlsHostReady()
        if (controlsHostEl.value && controlsHostProbeTimer) {
          clearInterval(controlsHostProbeTimer)
          controlsHostProbeTimer = null
        }
      }, 250)
    }
  }
)

function queueSearch() {
  if (debounceTimer) {
    clearTimeout(debounceTimer)
  }
  debounceTimer = setTimeout(() => {
    runSearch()
  }, 300)
}

function onWeightInput(weight: WeightItem) {
  weight.value = clamp(weight.value, 0, 1)
  if (props.searchFile) {
    needsConfirmSearch.value = true
  }
}

async function submitPromptForWeights() {
  const prompt = promptText.value.trim()
  if (!prompt || isPromptLoading.value) return

  isPromptLoading.value = true
  promptError.value = ''

  try {
    const response = await fetch(`${props.promptApiBase}/prompt_weights`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ prompt })
    })

    if (!response.ok) {
      throw new Error(await response.text())
    }

    const data = (await response.json()) as {
      structure: number
      color: number
      texture: number
    }

    applyPromptWeights(data)
  } catch (error) {
    promptError.value = `权重生成失败：${error instanceof Error ? error.message : '未知错误'}`
  } finally {
    isPromptLoading.value = false
  }
}

function applyPromptWeights(weights: { structure: number; color: number; texture: number }) {
  const nextValues: Record<WeightKey, number> = {
    structure: clamp(Number(weights.structure), 0, 1),
    color: clamp(Number(weights.color), 0, 1),
    texture: clamp(Number(weights.texture), 0, 1)
  }

  weightRows.value = weightRows.value.map((weight) => ({
    ...weight,
    value: nextValues[weight.key]
  }))

  if (props.searchFile) {
    needsConfirmSearch.value = true
  }
}

function onRepresentativeEnter(item: SearchItem) {
  if (pinnedItem.value) return
  hoverItem.value = item
}

function onRepresentativeLeave() {
  if (pinnedItem.value) return
  hoverItem.value = null
}

function onRepresentativeClick(item: SearchItem) {
  pinnedItem.value = item
  resetPreviewTransform()
}

function clearPinnedPreview() {
  pinnedItem.value = null
  hoverItem.value = null
  resetPreviewTransform()
}

function onShadowEnter(item: SearchItem) {
  if (pinnedItem.value) return
  hoverItem.value = item
}

function onShadowLeave() {
  if (pinnedItem.value) return
  hoverItem.value = null
}

function onShadowClick(item: SearchItem) {
  pinnedItem.value = item
  resetPreviewTransform()
}

function addPinnedToGallery() {
  if (!pinnedItem.value) return
  const species = pinnedItem.value.label.trim()
  const item: GalleryItem = {
    id: `${Date.now()}-${Math.random().toString(36).slice(2, 8)}`,
    imageUrl: resolveImageUrl(pinnedItem.value.path),
    species
  }
  galleryUploads.value.unshift(item)
  syncSelectedSpeciesFromGallery()
}

function removeGalleryItem(itemId: string) {
  galleryUploads.value = galleryUploads.value.filter((item) => item.id !== itemId)
  syncSelectedSpeciesFromGallery()
}

function syncSelectedSpeciesFromGallery() {
  const deduped = collectGallerySpecies()
  needsConfirmSpecies.value = !isSameSpeciesList(deduped, selectedSpecies.value)
}

function onConfirmGallerySpecies() {
  const deduped = collectGallerySpecies()
  selectedSpecies.value = deduped
  needsConfirmSpecies.value = false
  emit('species-change', [...deduped])
}

function collectGallerySpecies() {
  const deduped: string[] = []
  for (const item of galleryUploads.value) {
    const sp = item.species.trim()
    if (sp && !deduped.includes(sp)) {
      deduped.push(sp)
    }
  }
  return deduped
}

function isSameSpeciesList(a: string[], b: string[]) {
  if (a.length !== b.length) return false
  return a.every((value, index) => value === b[index])
}

function onOrbitWheel(event: WheelEvent) {
  if (!orbitRef.value) return

  const factor = event.deltaY < 0 ? 1.1 : 0.9
  orbitScale.value = clamp(orbitScale.value * factor, 0.6, 3)
}

function onOrbitMouseDown(event: MouseEvent) {
  if (event.button !== 0) return
  isPanning.value = true
  panStartX = event.clientX
  panStartY = event.clientY
  panOriginX = orbitOffsetX.value
  panOriginY = orbitOffsetY.value
}

function onOrbitMouseMove(event: MouseEvent) {
  if (!isPanning.value) return
  orbitOffsetX.value = panOriginX + (event.clientX - panStartX)
  orbitOffsetY.value = panOriginY + (event.clientY - panStartY)
}

function onOrbitMouseUp() {
  isPanning.value = false
}

function onConfirmWeights() {
  if (!props.searchFile || isSearching.value) return
  needsConfirmSearch.value = false
  queueSearch()
}

function isSpeciesHighlighted(item: SearchItem) {
  const species = highlightedSpecies.value
  if (!species) return true
  return item.label === species
}

function onPreviewWheel(event: WheelEvent) {
  const factor = event.deltaY < 0 ? 1.1 : 0.9
  imageScale.value = clamp(imageScale.value * factor, 1, 6)
  if (imageScale.value <= 1.02) {
    resetPreviewTransform()
  }
}

function onPreviewMouseDown(event: MouseEvent) {
  if (event.button !== 0 || imageScale.value <= 1) return
  isImageDragging.value = true
  imageDragStartX = event.clientX
  imageDragStartY = event.clientY
  imageDragOriginX = imageOffsetX.value
  imageDragOriginY = imageOffsetY.value
}

function onPreviewMouseMove(event: MouseEvent) {
  if (!isImageDragging.value) return
  imageOffsetX.value = imageDragOriginX + (event.clientX - imageDragStartX)
  imageOffsetY.value = imageDragOriginY + (event.clientY - imageDragStartY)
}

function onPreviewMouseUp() {
  isImageDragging.value = false
}

function zoomInPreview() {
  imageScale.value = clamp(imageScale.value * 1.2, 1, 6)
}

function zoomOutPreview() {
  imageScale.value = clamp(imageScale.value / 1.2, 1, 6)
  if (imageScale.value <= 1.02) {
    resetPreviewTransform()
  }
}

function resetPreviewTransform() {
  imageScale.value = 1
  imageOffsetX.value = 0
  imageOffsetY.value = 0
  isImageDragging.value = false
}

async function runSearch() {
  if (!props.searchFile) return

  if (activeController) {
    activeController.abort()
  }

  const controller = new AbortController()
  activeController = controller
  isSearching.value = true
  searchError.value = ''
  hoverItem.value = null

  const formData = new FormData()
  formData.append('file', props.searchFile)
  formData.append('structure_w', String(getWeightValue('structure')))
  formData.append('color_w', String(getWeightValue('color')))
  formData.append('texture_w', String(getWeightValue('texture')))
  formData.append('include_all_members', '1')
  formData.append('members_per_species', String(SHADOW_POINTS_PER_SPECIES))
  formData.append('nearby_k', '20')
  formData.append('max_total', '3000')

  try {
    console.log('Sending search request with weights:', {
      structure: getWeightValue('structure'),
      color: getWeightValue('color'),
      texture: getWeightValue('texture')
    })
    const response = await fetch(`${props.searchApiBase}/photo_search`, {
      method: 'POST',
      body: formData,
      signal: controller.signal
    })

    if (!response.ok) {
      throw new Error(await response.text())
    }

    const data = (await response.json()) as SearchItem[]
    console.log('Search results:', data)
    searchResults.value = data
    needsConfirmSearch.value = false

    if (pinnedItem.value) {
      const stillExists = data.some((item) => item.path === pinnedItem.value?.path)
      if (!stillExists) {
        pinnedItem.value = null
      }
    }
  } catch (error) {
    if (controller.signal.aborted) return
    searchResults.value = []
    searchError.value = `检索失败：${error instanceof Error ? error.message : '未知错误'}`
  } finally {
    if (activeController === controller) {
      activeController = null
    }
    isSearching.value = false
  }
}

function getWeightValue(key: WeightKey) {
  return weightRows.value.find((weight) => weight.key === key)?.value ?? 0
}

function toOrbitPoint(item: SearchItem): OrbitPoint {
  const rawRadius = Math.max(0, item.radius)
  const minR = orbitRadiusStats.value.min
  const maxR = orbitRadiusStats.value.max
  const span = Math.max(1e-6, maxR - minR)
  let normalized = (rawRadius - minR) / span

  // 如果后端半径过于集中，加入稳定扰动，避免整圈都挤在外缘。
  if (maxR - minR < 0.08) {
    const seed = hashSeed(item.path)
    const pseudo = (seed % 1000) / 1000
    normalized = clamp(0.15 + pseudo * 0.7, 0, 1)
  }

  const n = clamp(normalized, 0, 1)
  // 指数级径向映射：越往外半径增长越快，外圈更分散。
  const expK = 2.8
  const eased = Math.expm1(expK * n) / Math.expm1(expK)
  const radius = 6 + eased * 43
  const radian = (item.angle * Math.PI) / 180
  const point = clampToOrbitCircle(50 + Math.cos(radian) * radius, 50 + Math.sin(radian) * radius, 48)
  return {
    x: point.x,
    y: point.y,
    raw: item
  }
}

function pullShadowPointTowardAnchor(
  base: OrbitPoint,
  anchor: { x: number; y: number },
  seedSource: string
) {
  const seed = hashSeed(seedSource)
  const baseAngle = Math.atan2(base.y - anchor.y, base.x - anchor.x)
  const seedAngle = ((seed % 360) * Math.PI) / 180
  const finalAngle = baseAngle * 0.35 + seedAngle * 0.65
  const clusterRadius = 2.8 + ((seed >>> 5) % 1000) / 1000 * 4.8

  const mixedX = anchor.x + Math.cos(finalAngle) * clusterRadius
  const mixedY = anchor.y + Math.sin(finalAngle) * clusterRadius

  const clamped = clampToOrbitCircle(mixedX, mixedY, 48.6)
  return {
    x: clamped.x,
    y: clamped.y
  }
}

function clampToOrbitCircle(x: number, y: number, limit = 45) {
  const dx = x - 50
  const dy = y - 50
  const dist = Math.hypot(dx, dy)
  if (dist <= limit || dist < 1e-6) {
    return {
      x: clamp(x, 1, 99),
      y: clamp(y, 1, 99)
    }
  }

  const ratio = limit / dist
  return {
    x: 50 + dx * ratio,
    y: 50 + dy * ratio
  }
}

function hashSeed(text: string) {
  let hash = 2166136261
  for (let i = 0; i < text.length; i += 1) {
    hash ^= text.charCodeAt(i)
    hash += (hash << 1) + (hash << 4) + (hash << 7) + (hash << 8) + (hash << 24)
  }
  return hash >>> 0
}

function resolveImageUrl(path: string) {
  if (/^https?:\/\//i.test(path)) {
    return path
  }

  const base = props.searchApiBase.replace(/\/$/, '')
  let normalized = path.replace(/\\/g, '/')

  // 兼容绝对磁盘路径，优先截取 /data/... 段用于后端静态映射。
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

  // 编码中文与空格等字符，避免部分图片 404。
  const encodedPath = normalized
    .split('/')
    .map((segment, idx) => (idx === 0 ? segment : encodeURIComponent(segment)))
    .join('/')

  return `${base}${encodedPath}`
}

function clamp(value: number, min: number, max: number) {
  return Math.max(min, Math.min(max, value))
}
</script>

<template>
  <section class="panel panel-compare">
    <Teleport v-if="useExternalControls" :to="controlsHostEl!">
      <div class="teleported-controls">
        <div class="prompt-card">
          <div class="prompt-header">
            <span class="prompt-search-icon" aria-hidden="true">🔍</span>
            <span class="prompt-title">Prompt</span>
          </div>
          <div class="prompt-input">
            <slot name="prompt">
              <div class="prompt-input-wrap">
              <textarea
                v-model="promptText"
                :placeholder="props.promptPlaceholder"
                aria-label="prompt"
                rows="4"
              ></textarea>
                <button
                  class="prompt-send-btn"
                  type="button"
                  :disabled="isPromptLoading || !promptText.trim()"
                  @click="submitPromptForWeights"
                >
                  {{ isPromptLoading ? '...' : '➤' }}
                </button>
              </div>
            </slot>
          </div>
          <div v-if="promptError" class="prompt-error">{{ promptError }}</div>
          <div class="weights">
            <div class="weights-title">Field weights (0-1)</div>
            <slot name="fieldweights">
              <div class="weights-list">
                <div
                  v-for="weight in weightRows"
                  :key="weight.key"
                  class="weight-row"
                >
                  <span>{{ weight.label }}</span>
                  <input
                    v-model.number="weight.value"
                    :class="['weight-slider', `tone-${weight.tone ?? 'mint'}`]"
                    type="range"
                    min="0"
                    max="1"
                    step="0.01"
                    @input="onWeightInput(weight)"
                  />
                  <span class="weight-value">{{ weight.value.toFixed(2) }}</span>
                </div>
              </div>
            </slot>
            <div class="weights-actions">
              <button
                class="weights-confirm-btn"
                type="button"
                :disabled="!props.searchFile || isSearching"
                @click="onConfirmWeights"
              >
                确定
              </button>
            </div>
          </div>
        </div>
      </div>
    </Teleport>

    <div class="compare-grid" :class="{ 'compare-grid--no-controls': useExternalControls }">
      <div v-if="!useExternalControls" class="compare-column compare-column-controls">
        <div class="prompt-card">
          <div class="prompt-header">
            <span class="prompt-search-icon" aria-hidden="true">🔍</span>
            <span class="prompt-title">Prompt</span>
          </div>
          <div class="prompt-input">
            <slot name="prompt">
              <div class="prompt-input-wrap">
              <textarea
                v-model="promptText"
                :placeholder="props.promptPlaceholder"
                aria-label="prompt"
                rows="4"
              ></textarea>
                <button
                  class="prompt-send-btn"
                  type="button"
                  :disabled="isPromptLoading || !promptText.trim()"
                  @click="submitPromptForWeights"
                >
                  {{ isPromptLoading ? '...' : '➤' }}
                </button>
              </div>
            </slot>
          </div>
          <div v-if="promptError" class="prompt-error">{{ promptError }}</div>
          <div class="weights">
            <div class="weights-title">Field weights (0-1)</div>
            <slot name="fieldweights">
              <div class="weights-list">
                <div
                  v-for="weight in weightRows"
                  :key="weight.key"
                  class="weight-row"
                >
                  <span>{{ weight.label }}</span>
                  <input
                    v-model.number="weight.value"
                    :class="['weight-slider', `tone-${weight.tone ?? 'mint'}`]"
                    type="range"
                    min="0"
                    max="1"
                    step="0.01"
                    @input="onWeightInput(weight)"
                  />
                  <span class="weight-value">{{ weight.value.toFixed(2) }}</span>
                </div>
              </div>
            </slot>
            <div class="weights-actions">
              <button
                class="weights-confirm-btn"
                type="button"
                :disabled="!props.searchFile || isSearching"
                @click="onConfirmWeights"
              >
                确定
              </button>
            </div>
          </div>
        </div>
      </div>

      <div class="compare-column">
        <slot name="compare">
          <div
            ref="orbitRef"
            class="orbit"
            @mouseleave="onRepresentativeLeave"
            @wheel.prevent="onOrbitWheel"
            @mousedown="onOrbitMouseDown"
            @mousemove="onOrbitMouseMove"
            @mouseup="onOrbitMouseUp"
            @mouseleave.self="onOrbitMouseUp"
          >
            <div v-if="!props.searchFile" class="orbit-state">
              左侧编辑完成后点击“暂存”，并在 Field weights 点击“确定”生成轨道图
            </div>
            <div v-else-if="isSearching" class="orbit-state orbit-state-loading">
              <span class="loading-spinner" aria-hidden="true"></span>
              <span>正在计算轨道图...</span>
            </div>
            <div v-else-if="needsConfirmSearch" class="orbit-state">
              已更新 Field weights，点击右侧“确定”后开始计算轨道图
            </div>
            <div v-else-if="searchError" class="orbit-state orbit-state-error">{{ searchError }}</div>
            <div v-else-if="!searchResults.length" class="orbit-state">暂无检索结果</div>
            <div class="orbit-scene" :style="orbitSceneStyle">
              <svg
                v-if="searchResults.length"
                class="orbit-chart"
                viewBox="0 0 100 100"
                role="img"
                aria-label="search orbit"
              >
                <circle class="orbit-ring" cx="50" cy="50" r="15" />
                <circle class="orbit-ring" cx="50" cy="50" r="30" />
                <circle class="orbit-ring" cx="50" cy="50" r="44" />
              </svg>

              <button
                v-for="(point, index) in shadowPoints"
                :key="`${point.raw.path}-${index}`"
                type="button"
                class="shadow-node"
                :class="{ 'is-dimmed': !isSpeciesHighlighted(point.raw), 'is-active': highlightedSpecies === point.raw.label }"
                :style="{ left: `${point.x}%`, top: `${point.y}%`, width: `${shadowRadius * 2}%`, height: `${shadowRadius * 2}%` }"
                @mousedown.stop
                @mouseenter="onShadowEnter(point.raw)"
                @mouseleave="onShadowLeave"
                @click="onShadowClick(point.raw)"
                :aria-label="point.raw.label"
              >
                <span class="shadow-dot"></span>
              </button>

              <div v-if="orbitCenterImageUrl" class="orbit-center-image">
                <img :src="orbitCenterImageUrl" alt="center" />
              </div>

              <button
                v-for="(node, index) in representativeNodes"
                :key="`${node.raw.path}-rep-${index}`"
                type="button"
                class="rep-node"
                :class="{ 'is-dimmed': !isSpeciesHighlighted(node.raw), 'is-active': highlightedSpecies === node.raw.label }"
                :style="{ left: `${node.x}%`, top: `${node.y}%` }"
                @mousedown.stop
                @mouseenter="onRepresentativeEnter(node.raw)"
                @mouseleave="onRepresentativeLeave"
                @click="onRepresentativeClick(node.raw)"
              >
                <svg class="circle-arcs circle-arcs-node" viewBox="0 0 100 100" aria-hidden="true">
                  <g transform="rotate(-130 50 50)">
                    <circle
                      class="weight-arc-track"
                      cx="50"
                      cy="50"
                      r="47"
                    />
                    <circle
                      class="weight-arc arc-structure"
                      cx="50"
                      cy="50"
                      r="47"
                      pathLength="1"
                      :stroke-dasharray="`${arcStroke.structure} 1`"
                    />
                  </g>
                  <g transform="rotate(15 50 50)">
                    <circle
                      class="weight-arc-track"
                      cx="50"
                      cy="50"
                      r="49"
                    />
                    <circle
                      class="weight-arc arc-color"
                      cx="50"
                      cy="50"
                      r="49"
                      pathLength="1"
                      :stroke-dasharray="`${arcStroke.color} 1`"
                    />
                  </g>
                  <g transform="rotate(75 50 50)">
                    <circle
                      class="weight-arc-track"
                      cx="50"
                      cy="50"
                      r="51"
                    />
                    <circle
                      class="weight-arc arc-texture"
                      cx="50"
                      cy="50"
                      r="51"
                      pathLength="1"
                      :stroke-dasharray="`${arcStroke.texture} 1`"
                    />
                  </g>
                </svg>
                <img
                  :src="node.imageUrl"
                  :alt="node.raw.label"
                  loading="lazy"
                  decoding="async"
                />
              </button>
            </div>

            <div v-if="pinnedItem" class="pin-tip">
              已固定: {{ pinnedItem.label }}
              <button type="button" class="pin-clear" @click="clearPinnedPreview">取消固定</button>
            </div>
          </div>
        </slot>
      </div>

      <div class="compare-column compare-column-media">
        <div class="panel-block">
          <div class="panel-subtitle-row">
            <div class="panel-subtitle">Image</div>
            <button
              class="upload-icon-btn"
              type="button"
              title="加入 Gallery"
              :disabled="!pinnedItem"
              @click="addPinnedToGallery"
            >
              <img class="upload-icon-img" :src="uploadIconUrl" alt="upload" />
            </button>
          </div>
          <slot name="image">
            <div
              class="hero-image"
              @wheel.prevent="onPreviewWheel"
              @mousedown="onPreviewMouseDown"
              @mousemove="onPreviewMouseMove"
              @mouseup="onPreviewMouseUp"
              @mouseleave="onPreviewMouseUp"
            >
              <img
                v-if="displayImageUrl"
                :src="displayImageUrl"
                alt="reference"
                :style="previewImageStyle"
                :class="{ 'is-draggable': imageScale > 1 }"
              />
              <div v-else class="hero-empty">等待左侧保存后的分割图</div>
              <div v-if="displayImageUrl" class="image-zoom-tools">
                <button type="button" class="image-zoom-btn" @click="zoomInPreview">+</button>
                <button type="button" class="image-zoom-btn" @click="zoomOutPreview">-</button>
                <button type="button" class="image-zoom-btn" @click="resetPreviewTransform">1:1</button>
              </div>
            </div>
            <div v-if="activePreviewItem" class="image-meta">
              {{ activePreviewItem.label }} · 相似度 {{ activePreviewItem.score.toFixed(4) }}
            </div>
          </slot>
        </div>
        <div class="panel-block gallery-block">
          <div class="panel-subtitle">Gallery</div>
          <slot name="gallery">
            <div class="gallery-grid">
              <div v-if="!galleryResultImages.length" class="gallery-empty">暂无图片，点击 Image 右上角上传</div>
              <div
                v-for="(item, index) in galleryResultImages"
                :key="item.id"
                class="gallery-item"
              >
                <img
                  :src="item.imageUrl"
                  :alt="`thumb ${index + 1}`"
                />
                <button
                  class="gallery-remove-btn"
                  type="button"
                  title="移除"
                  @click="removeGalleryItem(item.id)"
                >
                  ×
                </button>
              </div>
            </div>
            <div class="gallery-actions">
              <button
                class="gallery-confirm-btn"
                type="button"
                :disabled="!needsConfirmSpecies"
                @click="onConfirmGallerySpecies"
              >
                确认
              </button>
            </div>
          </slot>
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
  padding: 18px;
  box-shadow: var(--shadow-soft);
}

.panel-compare {
  display: flex;
  flex-direction: column;
  gap: 16px;
  min-height: 0;
  overflow: hidden;
}

.compare-grid {
  display: grid;
  grid-template-columns: 0.56fr 1.56fr 0.72fr;
  gap: 16px;
  align-items: stretch;
  flex: 1;
  min-height: 0;
}

.compare-grid--no-controls {
  grid-template-columns: 1.56fr 0.72fr;
}

.teleported-controls {
  height: 100%;
  min-height: 0;
}

.compare-column {
  display: flex;
  flex-direction: column;
  gap: 16px;
  min-height: 0;
}

.compare-column-controls {
  width: 100%;
}

.prompt-card {
  display: flex;
  flex-direction: column;
  gap: 10px;
  min-height: 0;
  flex: 1;
}

.prompt-header {
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: 700;
  color: #5f6b62;
}

.prompt-search-icon {
  font-size: 14px;
  line-height: 1;
}

.prompt-title {
  font-size: 14px;
  letter-spacing: 0.04em;
  text-transform: uppercase;
}

.prompt-input {
  padding: 8px 10px;
  border-radius: 12px;
  border: 1px solid var(--panel-border);
  background: #f6f8f3;
}

.prompt-input textarea {
  width: 100%;
  border: none;
  background: transparent;
  outline: none;
  font-size: 14px;
  line-height: 1.4;
  resize: none;
  min-height: 5.6em;
  font-family: inherit;
}

.prompt-input-wrap {
  position: relative;
}

.prompt-input-wrap textarea {
  padding-right: 38px;
}

.prompt-send-btn {
  position: absolute;
  right: 4px;
  bottom: 4px;
  width: 28px;
  height: 28px;
  border: 1px solid var(--panel-border);
  border-radius: 8px;
  background: #ffffff;
  color: #4f5c56;
  cursor: pointer;
}

.prompt-send-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.prompt-error {
  font-size: 12px;
  color: #a45448;
}

.weights-title {
  font-weight: 600;
  color: #6a756c;
  margin-bottom: 6px;
}

.weights {
  min-height: 0;
  flex: 1;
  display: flex;
  flex-direction: column;
  margin-top: 14px;
}

.weights-actions {
  margin-top: 14px;
  display: flex;
  justify-content: flex-end;
}

.weights-confirm-btn {
  border: 1px solid var(--panel-border);
  border-radius: 9px;
  background: #ffffff;
  color: #3f524c;
  font-weight: 700;
  font-size: 12px;
  padding: 7px 14px;
  cursor: pointer;
}

.weights-confirm-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.weights-list {
  min-height: 0;
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 22px;
}

.weight-row {
  display: grid;
  grid-template-columns: 72px 1fr 40px;
  align-items: center;
  gap: 12px;
  font-size: 13px;
  padding: 2px 0;
}

.weight-slider {
  width: 100%;
}

.weight-slider.tone-mint {
  accent-color: var(--accent-mint);
}

.weight-slider.tone-blue {
  accent-color: var(--accent-blue);
}

.weight-slider.tone-pink {
  accent-color: var(--accent-pink);
}

.weight-value {
  color: #6d7b73;
  font-weight: 600;
  text-align: right;
}

.orbit {
  position: relative;
  flex: none;
  border-radius: 50%;
  border: 1px dashed #d7ded4;
  background: radial-gradient(circle at center, #f4f7f2 0%, #fdfdfb 50%, #f4f7f2 100%);
  aspect-ratio: 1 / 1;
  height: 100%;
  width: auto;
  max-width: 100%;
  align-self: center;
  min-height: 0;
  overflow: hidden;
}

.orbit-chart {
  width: 100%;
  height: 100%;
  display: block;
}

.orbit-scene {
  position: absolute;
  inset: 0;
  transform-origin: 50% 50%;
  will-change: transform;
}

.orbit-ring {
  fill: none;
  stroke: #dfe6dd;
  stroke-width: 0.48;
}

.orbit-shadow {
  fill: rgba(141, 151, 146, 0.4);
}

.shadow-node {
  position: absolute;
  transform: translate(-50%, -50%);
  border: none;
  background: transparent;
  border-radius: 50%;
  padding: 0;
  display: grid;
  place-items: center;
  cursor: pointer;
  transition: opacity 0.15s ease;
}

.shadow-dot {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  background: rgba(123, 133, 129, 0.72);
  border: 1px solid rgba(255, 255, 255, 0.45);
}

.shadow-node.is-dimmed {
  opacity: 0.28;
}

.shadow-node.is-active .shadow-dot {
  background: rgba(79, 164, 126, 0.95);
  border-color: rgba(255, 255, 255, 0.9);
}

.orbit-state {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  gap: 10px;
  text-align: center;
  padding: 0 24px;
  color: #6c776f;
  font-size: 13px;
}

.loading-spinner {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  border: 2px solid rgba(90, 108, 100, 0.24);
  border-top-color: #4f6b62;
  animation: orbit-spin 0.75s linear infinite;
}

@keyframes orbit-spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.orbit-state-error {
  color: #a45448;
}

.rep-node {
  position: absolute;
  width: 42px;
  height: 42px;
  transform: translate(-50%, -50%);
  border-radius: 50%;
  overflow: visible;
  border: none;
  box-shadow: none;
  cursor: pointer;
  padding: 0;
  background: transparent;
  transition: opacity 0.15s ease;
}

.rep-node.is-dimmed {
  opacity: 0.3;
}

.rep-node.is-active {
  opacity: 1;
}

.orbit-center-image {
  position: absolute;
  left: 50%;
  top: 50%;
  width: 44px;
  height: 44px;
  transform: translate(-50%, -50%);
  border-radius: 50%;
  overflow: visible;
  border: none;
  box-shadow: none;
  background: transparent;
}

.orbit-center-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: 50%;
}

.rep-node img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  image-rendering: auto;
  border-radius: 50%;
  transform: translateZ(0);
  backface-visibility: hidden;
}

.circle-arcs {
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  overflow: visible;
  pointer-events: none;
}

.circle-arcs-node {
  width: 52px;
  height: 52px;
}

.weight-arc {
  fill: none;
  stroke-linecap: round;
  vector-effect: non-scaling-stroke;
}

.weight-arc-track {
  fill: none;
  stroke: rgba(132, 141, 137, 0.28);
  stroke-width: 1.25;
  vector-effect: non-scaling-stroke;
}

.arc-structure {
  stroke: #d88a2e;
  stroke-width: 1.8;
}

.arc-color {
  stroke: #6a67dc;
  stroke-width: 1.8;
}

.arc-texture {
  stroke: #31d6be;
  stroke-width: 1.8;
}

.pin-tip {
  position: absolute;
  left: 12px;
  bottom: 12px;
  display: inline-flex;
  align-items: center;
  gap: 8px;
  background: rgba(255, 255, 255, 0.92);
  border: 1px solid var(--panel-border);
  border-radius: 999px;
  padding: 6px 10px;
  font-size: 12px;
  color: #4b5953;
}

.pin-clear {
  border: none;
  background: transparent;
  color: var(--accent-blue);
  font-size: 12px;
  cursor: pointer;
}

.panel-block {
  display: flex;
  flex-direction: column;
  gap: 12px;
  min-height: 0;
}

.compare-column-media {
  padding-bottom: 6px;
}

.gallery-block {
  flex: 1;
}

.panel-subtitle {
  font-weight: 700;
  font-size: 14px;
  margin-bottom: 12px;
  letter-spacing: 0.06em;
  color: #6a756c;
}

.panel-subtitle-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.upload-icon-btn {
  width: 28px;
  height: 28px;
  border-radius: 8px;
  border: 1px solid var(--panel-border);
  background: #ffffff;
  color: #4f5c56;
  font-size: 14px;
  cursor: pointer;
}

.upload-icon-img {
  width: 16px;
  height: 16px;
  object-fit: contain;
  display: block;
  margin: 0 auto;
}

.upload-icon-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.hero-image {
  border-radius: 14px;
  overflow: hidden;
  height: 148px;
  border: 1px solid var(--panel-border);
  background: #f7faf5;
  position: relative;
  cursor: default;
}

.hero-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  image-rendering: auto;
  transform-origin: 50% 50%;
}

.hero-image img.is-draggable {
  cursor: grab;
}

.image-zoom-tools {
  position: absolute;
  right: 8px;
  top: 8px;
  display: flex;
  gap: 6px;
}

.image-zoom-btn {
  border: 1px solid rgba(255, 255, 255, 0.9);
  background: rgba(40, 52, 48, 0.64);
  color: #ffffff;
  border-radius: 7px;
  font-size: 11px;
  font-weight: 700;
  height: 22px;
  min-width: 22px;
  padding: 0 6px;
  cursor: pointer;
}

.hero-empty {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #6f7b74;
  font-size: 12px;
  text-align: center;
  padding: 10px;
}

.image-meta {
  font-size: 12px;
  color: #617068;
}

.gallery-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 10px;
  min-height: 0;
  overflow-y: auto;
  padding-right: 4px;
  padding-bottom: 8px;
  align-content: start;
}

.gallery-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 4px;
}

.gallery-confirm-btn {
  border: 1px solid var(--panel-border);
  border-radius: 9px;
  background: #ffffff;
  color: #3f524c;
  font-weight: 700;
  font-size: 12px;
  padding: 7px 14px;
  cursor: pointer;
}

.gallery-confirm-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.gallery-empty {
  grid-column: 1 / -1;
  font-size: 12px;
  color: #6f7b74;
  text-align: center;
  padding: 16px 8px;
  border: 1px dashed var(--panel-border);
  border-radius: 10px;
}

.gallery-grid img {
  border-radius: 10px;
  height: 80px;
  object-fit: cover;
}

.gallery-item {
  position: relative;
}

.gallery-item img {
  width: 100%;
}

.gallery-remove-btn {
  position: absolute;
  right: 6px;
  bottom: 6px;
  width: 18px;
  height: 18px;
  border-radius: 999px;
  border: 1px solid rgba(255, 255, 255, 0.85);
  background: rgba(24, 26, 26, 0.72);
  color: #ffffff;
  font-size: 12px;
  line-height: 1;
  display: grid;
  place-items: center;
  opacity: 0;
  pointer-events: none;
  cursor: pointer;
  transition: opacity 0.16s ease;
}

.gallery-item:hover .gallery-remove-btn {
  opacity: 1;
  pointer-events: auto;
}

@media (max-width: 1200px) {
  .compare-grid {
    grid-template-columns: 1fr;
  }

  .orbit {
    width: 100%;
    height: auto;
  }
}

@media (max-width: 720px) {
  .gallery-grid {
    grid-template-columns: repeat(3, 1fr);
  }
}
</style>
