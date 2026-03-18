<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import furongjinjituUrl from '../assets/furongjinjitu.jpg'
import meimotuUrl from '../assets/momeitu.jpg'
import zhuquetuUrl from '../assets/zhumeihanquetu.jpg'
import shanchajiUrl from '../assets/shanchajixuetu.jpg'

type PresetImageKey = 'furongjinjitu' | 'zhuquetu' | 'meimotu' | 'shanchajixuetu'

interface Props {
  backendBase?: string
  presetImage?: PresetImageKey
}

const props = withDefaults(defineProps<Props>(), {
  backendBase:
    typeof window !== 'undefined' ? `http://${window.location.hostname}:8000` : 'http://127.0.0.1:8000',
  presetImage: 'furongjinjitu'
})

const emit = defineEmits<{
  saved: [payload: { maskUrl: string; file: File }]
}>()

const fileInputRef = ref<HTMLInputElement | null>(null)
const canvasRef = ref<HTMLCanvasElement | null>(null)
const overlayRef = ref<HTMLImageElement | null>(null)
const viewportRef = ref<HTMLDivElement | null>(null)

const isPositive = ref(true)
const points = ref<Array<[number, number]>>([])
const labels = ref<number[]>([])
const statusText = ref('请上传图片后点击画面进行分割')
const isUploading = ref(false)
const isSegmenting = ref(false)
const originalImage = ref<HTMLImageElement | null>(null)
const overlayUrl = ref('')
const currentMaskBlob = ref<Blob | null>(null)
const savedMasks = ref<string[]>([])
const imageWidth = ref(0)
const imageHeight = ref(0)
const fitScale = ref(1)
const zoomScale = ref(1)
const offsetX = ref(0)
const offsetY = ref(0)
const isDragging = ref(false)

let dragStartX = 0
let dragStartY = 0
let dragOriginX = 0
let dragOriginY = 0
let dragMoved = false
let segmentAbortController: AbortController | null = null
let requestSerial = 0

const PRESET_IMAGE_CONFIG: Record<PresetImageKey, { url: string; fileName: string }> = {
  furongjinjitu: {
    url: furongjinjituUrl,
    fileName: 'furongjinjitu.jpg'
  },
  zhuquetu: {
    url: zhuquetuUrl,
    fileName: 'zhuquetu.jpg'
  },
  meimotu: {
    url: meimotuUrl,
    fileName: 'meimotu.jpg'
  },
  shanchajixuetu: {
    url: shanchajiUrl,
    fileName: 'shanchajixuetu.jpeg'
  }
}

const modeLabel = computed(() =>
  isPositive.value ? '模式：正交互 (选中)' : '模式：负交互 (排除)'
)

const busy = computed(() => isUploading.value)
const currentScale = computed(() => fitScale.value * zoomScale.value)
const hasMaskOverlay = computed(() => Boolean(overlayUrl.value))

const stageStyle = computed(() => ({
  width: `${imageWidth.value}px`,
  height: `${imageHeight.value}px`,
  transform: `translate(${offsetX.value}px, ${offsetY.value}px) scale(${currentScale.value})`
}))

function openFilePicker() {
  fileInputRef.value?.click()
}

async function onFileChange(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (!file) return

  await uploadAndInitFile(file)
}

async function uploadAndInitFile(file: File) {
  isUploading.value = true
  statusText.value = '正在上传并预计算特征...'

  const formData = new FormData()
  formData.append('file', file)

  try {
    const response = await fetch(`${props.backendBase}/upload`, {
      method: 'POST',
      body: formData
    })
    if (!response.ok) {
      throw new Error(await response.text())
    }
    const data = await response.json()
    await initCanvasByFile(file)
    resetPoints(false)
    resetViewTransform()
    statusText.value = `图片已加载 (${data.resolution})`
  } catch (error) {
    statusText.value = `上传失败：${error instanceof Error ? error.message : '未知错误'}`
  } finally {
    isUploading.value = false
  }
}

function initCanvasByFile(file: File) {
  return new Promise<void>((resolve, reject) => {
    const image = new Image()
    const objectUrl = URL.createObjectURL(file)
    image.onload = () => {
      const canvas = canvasRef.value
      if (!canvas) {
        URL.revokeObjectURL(objectUrl)
        reject(new Error('Canvas 未初始化'))
        return
      }

      canvas.width = image.width
      canvas.height = image.height
      imageWidth.value = image.width
      imageHeight.value = image.height
      const context = canvas.getContext('2d')
      if (!context) {
        URL.revokeObjectURL(objectUrl)
        reject(new Error('Canvas 2D 上下文不可用'))
        return
      }

      context.clearRect(0, 0, canvas.width, canvas.height)
      context.drawImage(image, 0, 0)
      originalImage.value = image
      URL.revokeObjectURL(objectUrl)
      nextTick(() => {
        resolve()
      })
    }
    image.onerror = () => {
      URL.revokeObjectURL(objectUrl)
      reject(new Error('图片解码失败'))
    }
    image.src = objectUrl
  })
}

async function onCanvasClick(event: MouseEvent) {
  if (!originalImage.value || isUploading.value) {
    if (!originalImage.value) {
      openFilePicker()
    }
    return
  }

  if (dragMoved) {
    dragMoved = false
    return
  }

  const viewport = viewportRef.value
  if (!viewport || !imageWidth.value || !imageHeight.value) return

  const rect = viewport.getBoundingClientRect()
  const vx = event.clientX - rect.left
  const vy = event.clientY - rect.top
  const x = Math.round((vx - offsetX.value) / currentScale.value)
  const y = Math.round((vy - offsetY.value) / currentScale.value)

  if (x < 0 || y < 0 || x >= imageWidth.value || y >= imageHeight.value) {
    return
  }

  points.value.push([x, y])
  labels.value.push(isPositive.value ? 1 : 0)
  drawPoints()
  requestSegment()
}

function onWheel(event: WheelEvent) {
  if (!originalImage.value || !viewportRef.value) return

  const viewport = viewportRef.value
  const rect = viewport.getBoundingClientRect()
  const cursorX = event.clientX - rect.left
  const cursorY = event.clientY - rect.top
  const previousScale = currentScale.value

  const factor = event.deltaY < 0 ? 1.1 : 0.9
  zoomScale.value = clamp(zoomScale.value * factor, 0.5, 8)
  const nextScale = currentScale.value

  if (nextScale === previousScale) return

  const imageX = (cursorX - offsetX.value) / previousScale
  const imageY = (cursorY - offsetY.value) / previousScale
  offsetX.value = cursorX - imageX * nextScale
  offsetY.value = cursorY - imageY * nextScale
}

function onPointerDown(event: MouseEvent) {
  if (!originalImage.value || event.button !== 0) return

  isDragging.value = true
  dragMoved = false
  dragStartX = event.clientX
  dragStartY = event.clientY
  dragOriginX = offsetX.value
  dragOriginY = offsetY.value
}

function onPointerMove(event: MouseEvent) {
  if (!isDragging.value) return

  const dx = event.clientX - dragStartX
  const dy = event.clientY - dragStartY
  if (Math.abs(dx) > 2 || Math.abs(dy) > 2) {
    dragMoved = true
  }
  offsetX.value = dragOriginX + dx
  offsetY.value = dragOriginY + dy
}

function onPointerUp() {
  isDragging.value = false
}

function resetViewTransform() {
  const viewport = viewportRef.value
  if (!viewport || !imageWidth.value || !imageHeight.value) return

  const viewportWidth = viewport.clientWidth
  const viewportHeight = viewport.clientHeight
  const scale = Math.min(viewportWidth / imageWidth.value, viewportHeight / imageHeight.value)

  fitScale.value = scale
  zoomScale.value = 1
  offsetX.value = (viewportWidth - imageWidth.value * scale) / 2
  offsetY.value = (viewportHeight - imageHeight.value * scale) / 2
}

function clamp(value: number, min: number, max: number) {
  return Math.max(min, Math.min(max, value))
}

function drawPoints() {
  const canvas = canvasRef.value
  if (!canvas || !originalImage.value) return
  const context = canvas.getContext('2d')
  if (!context) return

  context.clearRect(0, 0, canvas.width, canvas.height)
  context.drawImage(originalImage.value, 0, 0)

  points.value.forEach((point, index) => {
    context.fillStyle = labels.value[index] === 1 ? '#00B67A' : '#E2574C'
    context.beginPath()
    context.arc(point[0], point[1], 18, 0, Math.PI * 2)
    context.fill()
    context.lineWidth = 4
    context.strokeStyle = '#ffffff'
    context.stroke()
  })
}

async function requestSegment() {
  if (!points.value.length) return

  if (segmentAbortController) {
    segmentAbortController.abort()
  }

  const controller = new AbortController()
  segmentAbortController = controller
  const serial = ++requestSerial
  const pointsSnapshot = points.value.map((point) => [...point])
  const labelsSnapshot = [...labels.value]

  isSegmenting.value = true
  statusText.value = `SAM2 识别中... 点数 ${pointsSnapshot.length}`

  try {
    const response = await fetch(`${props.backendBase}/segment`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        points: pointsSnapshot,
        labels: labelsSnapshot
      }),
      signal: controller.signal
    })
    if (!response.ok) {
      throw new Error(await response.text())
    }

    if (serial !== requestSerial) {
      return
    }

    const blob = await response.blob()
    updateOverlay(blob)
    statusText.value = `分割成功，当前点数 ${labelsSnapshot.length}`
  } catch (error) {
    if (controller.signal.aborted) {
      return
    }
    if (serial !== requestSerial) {
      return
    }
    statusText.value = `分割失败：${error instanceof Error ? error.message : '未知错误'}`
  } finally {
    if (serial === requestSerial) {
      isSegmenting.value = false
      segmentAbortController = null
    }
  }
}

function updateOverlay(blob: Blob) {
  const overlay = overlayRef.value
  if (!overlay) return

  if (overlayUrl.value) {
    URL.revokeObjectURL(overlayUrl.value)
  }

  const nextUrl = URL.createObjectURL(blob)
  overlayUrl.value = nextUrl
  currentMaskBlob.value = blob
  overlay.src = nextUrl
  overlay.style.display = 'block'
}

function setPositiveMode() {
  isPositive.value = true
  statusText.value = modeLabel.value
}

function setNegativeMode() {
  isPositive.value = false
  statusText.value = modeLabel.value
}

function resetPoints(updateInfo = true) {
  if (segmentAbortController) {
    segmentAbortController.abort()
    segmentAbortController = null
  }

  points.value = []
  labels.value = []

  if (overlayRef.value) {
    overlayRef.value.style.display = 'none'
    overlayRef.value.src = ''
  }

  if (overlayUrl.value) {
    URL.revokeObjectURL(overlayUrl.value)
    overlayUrl.value = ''
  }

  currentMaskBlob.value = null
  isSegmenting.value = false

  if (originalImage.value) {
    drawPoints()
  }

  if (updateInfo) {
    statusText.value = '交互已重置'
  }
}

async function saveCurrentMask() {
  if (currentMaskBlob.value) {
    persistSavedBlob(currentMaskBlob.value)
    statusText.value = `已暂存到前端，共 ${savedMasks.value.length} 份结果`
    return
  }

  if (!canvasRef.value || !originalImage.value) {
    statusText.value = '暂无可保存内容，请先加载图片'
    return
  }

  const fallbackBlob = await canvasToBlob(canvasRef.value)
  if (!fallbackBlob) {
    statusText.value = '暂存失败：无法导出当前画布'
    return
  }

  persistSavedBlob(fallbackBlob)
  statusText.value = `已暂存当前画布并触发检索，共 ${savedMasks.value.length} 份结果`
}

function persistSavedBlob(blob: Blob) {
  const fileName = `edited-mask-${Date.now()}.png`
  const savedFile = new File([blob], fileName, { type: 'image/png' })
  const savedUrl = URL.createObjectURL(blob)
  savedMasks.value.push(savedUrl)
  emit('saved', {
    maskUrl: savedUrl,
    file: savedFile
  })
}

function canvasToBlob(canvas: HTMLCanvasElement): Promise<Blob | null> {
  return new Promise((resolve) => {
    canvas.toBlob((blob) => resolve(blob), 'image/png')
  })
}

onBeforeUnmount(() => {
  if (segmentAbortController) {
    segmentAbortController.abort()
  }
  if (overlayUrl.value) {
    URL.revokeObjectURL(overlayUrl.value)
  }
  savedMasks.value.forEach((url) => URL.revokeObjectURL(url))
})

onMounted(() => {
  void initializePresetImage()
})

watch(
  () => props.presetImage,
  (nextPreset, prevPreset) => {
    if (nextPreset !== prevPreset) {
      void initializePresetImage(true)
    }
  }
)

async function initializePresetImage(force = false) {
  if ((!force && originalImage.value) || isUploading.value) return

  const preset = PRESET_IMAGE_CONFIG[props.presetImage]

  try {
    const response = await fetch(preset.url)
    if (!response.ok) {
      throw new Error(`默认图加载失败: ${response.status}`)
    }
    const blob = await response.blob()
    const file = new File([blob], preset.fileName, { type: blob.type || 'image/jpeg' })
    await uploadAndInitFile(file)
  } catch (error) {
    statusText.value = `默认图初始化失败：${error instanceof Error ? error.message : '未知错误'}`
  }
}
</script>

<template>
  <section class="panel panel-left">
    <input
      ref="fileInputRef"
      class="hidden-file-input"
      type="file"
      accept="image/*"
      @change="onFileChange"
    />
    <div class="editor-split">
      <div class="editor-workspace">
        <div
          ref="viewportRef"
          class="editor-viewport"
          :class="{ 'is-dragging': isDragging, 'has-mask': hasMaskOverlay }"
          @wheel.prevent="onWheel"
          @mousedown="onPointerDown"
          @mousemove="onPointerMove"
          @mouseup="onPointerUp"
          @mouseleave="onPointerUp"
        >
          <div class="editor-stage" :style="stageStyle">
            <canvas
              ref="canvasRef"
              class="editor-canvas"
              @click="onCanvasClick"
            ></canvas>
            <img
              ref="overlayRef"
              class="mask-overlay"
              alt="segmentation mask"
            />
          </div>
          <div v-if="!originalImage" class="editor-placeholder">
            点击“上传图片”后开始交互分割
          </div>
        </div>
        <div class="tool-rail">
          <button
            class="tool-btn"
            :class="{ 'is-active': isPositive }"
            type="button"
            aria-label="正交互"
            title="正交互"
            @click="setPositiveMode"
          >
            ＋
          </button>
          <button
            class="tool-btn"
            :class="{ 'is-active': !isPositive }"
            type="button"
            aria-label="负交互"
            title="负交互"
            @click="setNegativeMode"
          >
            －
          </button>
          <button
            class="tool-btn"
            type="button"
            aria-label="清除"
            title="清除"
            @click="resetPoints()"
          >
            🗑
          </button>
          <button
            class="tool-btn"
            type="button"
            aria-label="暂存"
            title="暂存"
            @click="saveCurrentMask"
          >
            ✓
          </button>
        </div>

        <div class="editor-footer">
          <button
            class="upload-btn"
            type="button"
            :disabled="busy"
            @click="openFilePicker"
          >
            上传图片
          </button>
          <button
            class="upload-btn"
            type="button"
            :disabled="!originalImage"
            @click="resetViewTransform"
          >
            视图复位
          </button>
          <div class="editor-status">{{ statusText }}</div>
        </div>
      </div>
      <aside id="compare-controls-host" class="controls-dock"></aside>
    </div>
  </section>
</template>

<style scoped>
.panel {
  background: var(--panel-bg);
  border-radius: 18px;
  border: 1px solid var(--panel-border);
  box-shadow: var(--shadow-soft);
}

.panel-left {
  position: relative;
  display: block;
  min-height: 0;
  overflow: hidden;
  background: #d8ddd7;
  padding: 10px;
}

.editor-split {
  height: 100%;
  min-height: 0;
  display: grid;
  grid-template-columns: minmax(0, 1.65fr) minmax(250px, 1fr);
  gap: 10px;
}

.editor-workspace {
  position: relative;
  min-height: 0;
  border-radius: 12px;
  overflow: hidden;
}

.hidden-file-input {
  display: none;
}

.editor-viewport {
  position: absolute;
  inset: 0;
  overflow: hidden;
  border-radius: 12px;
}

.editor-stage {
  position: absolute;
  left: 0;
  top: 0;
  transform-origin: 0 0;
  will-change: transform;
}

.editor-canvas {
  width: 100%;
  height: 100%;
  display: block;
  cursor: grab;
  transition: filter 180ms ease;
}

.editor-viewport.has-mask .editor-canvas {
  filter: brightness(0.6) saturate(0.8) contrast(1.04);
}

.editor-viewport.is-dragging .editor-canvas {
  cursor: grabbing;
}

.mask-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 10;
  pointer-events: none;
  opacity: 0.98;
  display: none;
  mix-blend-mode: screen;
  filter: saturate(1.45) contrast(1.28) drop-shadow(0 0 10px rgba(0, 255, 120, 0.55));
}

.editor-placeholder {
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  z-index: 2;
  color: #5c6862;
  background: rgba(255, 255, 255, 0.85);
  border: 1px solid var(--panel-border);
  border-radius: 10px;
  padding: 8px 12px;
  font-size: 12px;
  pointer-events: none;
}

.tool-rail {
  position: absolute;
  left: 14px;
  top: 18px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.tool-btn {
  width: 32px;
  height: 32px;
  border-radius: 10px;
  border: 1px solid rgba(255, 255, 255, 0.7);
  background: rgba(255, 255, 255, 0.85);
  color: #425050;
  font-weight: 700;
  cursor: pointer;
}

.tool-btn.is-active {
  border-color: #86c9b3;
  background: #dff3ea;
  color: #245447;
}

.editor-footer {
  position: absolute;
  left: 12px;
  right: 12px;
  bottom: 12px;
  z-index: 12;
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 10px;
  border-radius: 10px;
  background: rgba(255, 255, 255, 0.56);
  backdrop-filter: blur(3px);
}

.upload-btn {
  border: 1px solid var(--panel-border);
  border-radius: 8px;
  background: #ffffff;
  color: #40524d;
  font-weight: 600;
  font-size: 12px;
  padding: 6px 10px;
  cursor: pointer;
  white-space: nowrap;
}

.upload-btn:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.editor-status {
  font-size: 12px;
  color: #4e625c;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}

.controls-dock {
  min-height: 0;
  min-width: 0;
  height: 100%;
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.5);
  border: 1px solid rgba(255, 255, 255, 0.6);
  padding: 10px;
  overflow: auto;
}

@media (max-width: 1200px) {
  .panel-left {
    min-height: 320px;
  }

  .editor-split {
    grid-template-columns: 1fr;
  }

  .controls-dock {
    min-height: 220px;
  }
}
</style>
