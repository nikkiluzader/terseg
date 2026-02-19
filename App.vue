<template>
  <div class="h-screen flex flex-col bg-slate-950 text-slate-100 overflow-hidden">
    <Header />

    <main class="flex-1 flex overflow-hidden">
      <!-- Left Control Bar -->
      <ControlPanel
        :params="params"
        :is-processing="isProcessing"
        :image-dimensions="imageDimensions"
        :road-settings="roadSettings"
        @update:params="params = $event"
        @update:road-settings="roadSettings = $event"
        @road-mask-file="loadRoadMask"
        @clear-road-mask="clearRoadMask"
      />

      <!-- Viewport Area -->
      <div class="flex-1 relative bg-slate-900 overflow-hidden flex flex-col">
        <!-- Hidden file input (always in DOM) -->
        <input type="file" class="hidden" accept="image/*" ref="fileInputEl" @change="handleFileUpload" />
          <!-- Viewport Toolbar -->
          <div class="absolute top-4 left-1/2 -translate-x-1/2 z-20 flex items-center gap-2 p-1.5 bg-slate-900/90 backdrop-blur-md border border-slate-700/50 rounded-2xl shadow-2xl" v-if="image">
            <button
              @click="openFilePicker"
              class="flex items-center gap-2 px-4 py-2 text-xs font-bold text-slate-300 hover:text-white hover:bg-slate-800 rounded-xl transition-all"
              title="Open new image"
            >
              <FolderOpen class="w-4 h-4" />
              Open
            </button>
            <div class="w-px h-6 bg-slate-700 mx-1" />
            <button
              @mousedown="showOriginal = true"
              @mouseup="showOriginal = false"
              @mouseleave="showOriginal = false"
              class="flex items-center gap-2 px-4 py-2 text-xs font-bold text-slate-300 hover:text-white hover:bg-slate-800 rounded-xl transition-all"
            >
              <component :is="showOriginal ? EyeOff : Eye" class="w-4 h-4" />
              Hold to Compare
            </button>
            <div class="w-px h-6 bg-slate-700 mx-1" />
            <button
              @click="handleDownload"
              class="flex items-center gap-2 px-4 py-2 text-xs font-bold bg-blue-600 hover:bg-blue-500 text-white rounded-xl transition-all"
            >
              <Download class="w-4 h-4" />
              Export
            </button>
            <div class="w-px h-6 bg-slate-700 mx-1" />
            <button
              @click="zoomIn"
              class="p-2 text-slate-300 hover:text-white hover:bg-slate-800 rounded-xl transition-all"
              title="Zoom In"
            >
              <ZoomIn class="w-4 h-4" />
            </button>
            <button
              @click="zoomOut"
              class="p-2 text-slate-300 hover:text-white hover:bg-slate-800 rounded-xl transition-all"
              title="Zoom Out"
            >
              <ZoomOut class="w-4 h-4" />
            </button>
            <button
              @click="resetView"
              class="p-2 text-slate-300 hover:text-white hover:bg-slate-800 rounded-xl transition-all"
              title="Reset View"
            >
              <Maximize2 class="w-4 h-4" />
            </button>
            <span class="text-[10px] font-mono text-slate-500 px-2 min-w-[3.5rem] text-center">{{ Math.round(zoom * 100) }}%</span>
          </div>

          <!-- Status Indicator -->
          <div
            v-if="image && clusterCount !== null"
            class="absolute bottom-6 left-6 z-20 px-4 py-2 bg-slate-900/80 backdrop-blur border border-slate-700 rounded-lg text-[11px] font-mono text-slate-400"
          >
            <div class="flex items-center gap-2">
              <div class="w-2 h-2 bg-green-500 rounded-full animate-pulse" />
              SEGMENTATION COMPLETE: {{ clusterCount }} UNIQUE PLOTS IDENTIFIED
            </div>
          </div>

          <!-- Canvas Container -->
          <div
            ref="viewportEl"
            class="flex-1 overflow-hidden bg-[#0a0f1a] relative"
            :style="{ cursor: image ? 'none' : 'default' }"
            @wheel.prevent="onWheel"
            @pointerdown="onPointerDown"
            @pointermove="onPointerMove"
            @pointerup="onPointerUp"
            @pointerleave="cursorVisible = false"
            @pointerenter="cursorVisible = true"
          >
            <div
              class="absolute top-0 left-0"
              :style="{
                transform: `translate(${panX}px, ${panY}px) scale(${zoom})`,
                transformOrigin: '0 0',
                willChange: 'transform'
              }"
            >
              <div class="relative shadow-2xl rounded-sm overflow-hidden border border-slate-800">
                <canvas
                  v-show="showOriginal"
                  ref="originalCanvasEl"
                  class="absolute inset-0 w-full h-full z-10 pointer-events-none"
                  style="image-rendering: pixelated"
                />
                <canvas
                  ref="canvasEl"
                  class="block"
                  style="image-rendering: pixelated"
                />
                <canvas
                  v-if="roadSettings.hasRoadMask"
                  ref="roadCanvasEl"
                  class="absolute inset-0 w-full h-full z-20 pointer-events-none"
                />
              </div>
            </div>

            <!-- Empty state: upload prompt -->
            <div
              v-if="!image"
              class="absolute inset-0 flex flex-col items-center justify-center z-10"
            >
              <button
                @click="openFilePicker"
                class="flex flex-col items-center gap-4 p-8 rounded-2xl border-2 border-dashed border-slate-700 hover:border-blue-500/50 hover:bg-slate-800/30 transition-all group cursor-pointer"
              >
                <div class="p-4 bg-slate-800 rounded-2xl group-hover:bg-blue-600/20 transition-colors">
                  <Upload class="w-8 h-8 text-blue-500" />
                </div>
                <div class="text-center">
                  <p class="text-sm text-slate-300 font-semibold">Open an image to begin</p>
                  <p class="text-xs text-slate-500 mt-1">PNG, JPG or GeoTIFF</p>
                </div>
              </button>
            </div>

            <!-- Crosshair -->
            <svg
              v-if="image && cursorVisible && !isProcessing"
              class="pointer-events-none absolute top-0 left-0 w-full h-full z-10"
            >
              <line :x1="cursorX" :y1="0" :x2="cursorX" :y2="'100%'" stroke="rgba(59,130,246,0.5)" stroke-width="1" />
              <line :x1="0" :y1="cursorY" :x2="'100%'" :y2="cursorY" stroke="rgba(59,130,246,0.5)" stroke-width="1" />
              <circle :cx="cursorX" :cy="cursorY" r="6" fill="none" stroke="rgba(59,130,246,0.8)" stroke-width="1.5" />
              <circle :cx="cursorX" :cy="cursorY" r="1.5" fill="rgba(59,130,246,0.9)" />
            </svg>

            <!-- Processing Progress Bar (non-blocking) -->
            <div
              v-if="isProcessing"
              class="absolute top-0 left-0 right-0 z-30"
            >
              <div class="h-1 bg-slate-800/80 overflow-hidden">
                <div
                  class="h-full bg-blue-500 transition-all duration-150 ease-out"
                  :style="{ width: Math.round(progress * 100) + '%' }"
                />
              </div>
            </div>
          </div>
      </div>
    </main>

    <!-- Footer -->
    <footer class="h-8 bg-slate-950 border-t border-slate-900 flex items-center justify-between px-6 text-[10px] text-slate-600 font-medium shrink-0">
      <div class="flex gap-4">
        <span>LAT: 0.0000°</span>
        <span>LNG: 0.0000°</span>
        <span>RES: 30CM/PX</span>
      </div>
      <div>
        TERSEG ENGINE V1.2
      </div>
    </footer>
  </div>
</template>

<script setup>
import { ref, watch, nextTick } from 'vue'
import Header from './components/Header.vue'
import ControlPanel from './components/ControlPanel.vue'
import { Upload, Download, Eye, EyeOff, Layers as LayersIcon, ZoomIn, ZoomOut, Maximize2, FolderOpen } from 'lucide-vue-next'
import { segmentImage } from './services/segmentation.js'

// Debounce delay for slider-driven reprocessing (ms)
const DEBOUNCE_MS = 350

const image = ref(null)
const isProcessing = ref(false)
const params = ref({
  numColors: 10,
  blurRadius: 6,
  blurPasses: 3,
  modeRadius: 3,
  modePasses: 3,
  minRegionSize: 100,
  maxSize: 512
})
const showOriginal = ref(false)
const clusterCount = ref(null)
const progress = ref(0)
const imageDimensions = ref(null)

// Road overlay state
const roadSettings = ref({
  hasRoadMask: false,
  opacity: 0.85,
  color: '#4a4a4a',
  width: 1
})
let roadMaskBinary = null  // Uint8Array: 1 = road, 0 = not road
let roadMaskW = 0
let roadMaskH = 0

const canvasEl = ref(null)
const originalCanvasEl = ref(null)
const roadCanvasEl = ref(null)
const viewportEl = ref(null)
const fileInputEl = ref(null)
let originalImage = null
let currentAbortController = null
let debounceTimer = null

// Max canvas dimension to keep browser stable
const MAX_CANVAS_DIM = 4096
let displayScale = 1 // ratio of canvas size to original image size

const zoom = ref(1)
const panX = ref(0)
const panY = ref(0)
let isPanning = false
let panStart = { x: 0, y: 0 }
let panOrigin = { x: 0, y: 0 }

const cursorX = ref(0)
const cursorY = ref(0)
const cursorVisible = ref(false)

const MIN_ZOOM = 0.05
const MAX_ZOOM = 40

function zoomToPoint(newZoom, cx, cy) {
  newZoom = Math.min(MAX_ZOOM, Math.max(MIN_ZOOM, newZoom))
  const scale = newZoom / zoom.value
  panX.value = cx - scale * (cx - panX.value)
  panY.value = cy - scale * (cy - panY.value)
  zoom.value = newZoom
}

function onWheel(e) {
  if (!image.value) return
  e.preventDefault()
  const factor = e.deltaY > 0 ? 0.9 : 1.1
  const rect = viewportEl.value.getBoundingClientRect()
  const cx = e.clientX - rect.left
  const cy = e.clientY - rect.top
  zoomToPoint(zoom.value * factor, cx, cy)
}

function onPointerDown(e) {
  if (e.button !== 0 || !image.value) return
  isPanning = true
  panStart = { x: e.clientX, y: e.clientY }
  panOrigin = { x: panX.value, y: panY.value }
  viewportEl.value?.setPointerCapture(e.pointerId)
}

function onPointerMove(e) {
  const rect = viewportEl.value.getBoundingClientRect()
  cursorX.value = e.clientX - rect.left
  cursorY.value = e.clientY - rect.top
  if (!isPanning) return
  panX.value = panOrigin.x + (e.clientX - panStart.x)
  panY.value = panOrigin.y + (e.clientY - panStart.y)
}

function onPointerUp(e) {
  isPanning = false
  viewportEl.value?.releasePointerCapture(e.pointerId)
}

function zoomIn() {
  const rect = viewportEl.value?.getBoundingClientRect()
  if (!rect) return
  zoomToPoint(zoom.value * 1.3, rect.width / 2, rect.height / 2)
}

function zoomOut() {
  const rect = viewportEl.value?.getBoundingClientRect()
  if (!rect) return
  zoomToPoint(zoom.value / 1.3, rect.width / 2, rect.height / 2)
}

function fitToViewport() {
  if (!viewportEl.value || !canvasEl.value) return
  const vp = viewportEl.value.getBoundingClientRect()
  const cw = canvasEl.value.width
  const ch = canvasEl.value.height
  if (!cw || !ch) return

  const pad = 48
  const fitZoom = Math.min((vp.width - pad) / cw, (vp.height - pad) / ch, 1)
  zoom.value = fitZoom
  panX.value = (vp.width - cw * fitZoom) / 2
  panY.value = (vp.height - ch * fitZoom) / 2
}

function resetView() {
  fitToViewport()
}

function handleFileUpload(e) {
  const file = e.target.files?.[0]
  if (!file) return

  // Reset state for new file
  clusterCount.value = null
  showOriginal.value = false

  const reader = new FileReader()
  reader.onload = (event) => {
    const dataUrl = event.target?.result
    image.value = dataUrl
  }
  reader.onerror = () => console.error('Failed to read file')
  reader.readAsDataURL(file)

  // Reset input so the same file can be re-selected
  e.target.value = ''
}

function openFilePicker() {
  fileInputEl.value?.click()
}

function scheduleProcess() {
  if (debounceTimer) clearTimeout(debounceTimer)
  debounceTimer = setTimeout(() => processImage(), DEBOUNCE_MS)
}

async function processImage() {
  if (!canvasEl.value || !originalImage) return

  // Cancel any in-flight processing
  if (currentAbortController) {
    currentAbortController.abort()
  }
  const controller = new AbortController()
  currentAbortController = controller

  isProcessing.value = true
  progress.value = 0
  try {
    // Redraw original image onto canvas before processing
    const cw = canvasEl.value.width
    const ch = canvasEl.value.height
    const ctx = canvasEl.value.getContext('2d')
    ctx.imageSmoothingEnabled = true
    ctx.imageSmoothingQuality = 'high'
    ctx.drawImage(originalImage, 0, 0, cw, ch)

    await new Promise(r => setTimeout(r, 20))

    const { clusters } = await segmentImage(
      canvasEl.value,
      params.value,
      (p) => { if (!controller.signal.aborted) progress.value = p },
      controller.signal
    )
    clusterCount.value = clusters

    // Re-draw road overlay on top after segmentation
    if (roadSettings.value.hasRoadMask && roadMaskBinary) {
      nextTick(() => drawRoadOverlay())
    }
  } catch (error) {
    if (error.name === 'AbortError') return // superseded by newer request
    console.error("Processing error:", error)
  } finally {
    if (!controller.signal.aborted) {
      isProcessing.value = false
      progress.value = 0
    }
  }
}

// Auto-reprocess when any parameter changes
watch(params, () => {
  if (originalImage) scheduleProcess()
}, { deep: true })

function handleDownload() {
  if (!canvasEl.value) return

  // Composite segmented + road overlay for export
  const exportWithRoads = (srcCanvas) => {
    if (roadSettings.value.hasRoadMask && roadCanvasEl.value) {
      const comp = document.createElement('canvas')
      comp.width = srcCanvas.width
      comp.height = srcCanvas.height
      const cCtx = comp.getContext('2d')
      cCtx.drawImage(srcCanvas, 0, 0)
      cCtx.drawImage(roadCanvasEl.value, 0, 0, comp.width, comp.height)
      return comp
    }
    return srcCanvas
  }

  // If canvas was capped, export at full original resolution
  if (displayScale < 1 && originalImage) {
    const fullCanvas = document.createElement('canvas')
    fullCanvas.width = originalImage.width
    fullCanvas.height = originalImage.height
    const fCtx = fullCanvas.getContext('2d')
    fCtx.imageSmoothingEnabled = false
    fCtx.drawImage(canvasEl.value, 0, 0, originalImage.width, originalImage.height)
    const final = exportWithRoads(fullCanvas)
    const link = document.createElement('a')
    link.download = `segmented-satellite-${Date.now()}.png`
    link.href = final.toDataURL('image/png')
    link.click()
  } else {
    const final = exportWithRoads(canvasEl.value)
    const link = document.createElement('a')
    link.download = `segmented-satellite-${Date.now()}.png`
    link.href = final.toDataURL('image/png')
    link.click()
  }
}

// ─── Road mask loading and rendering ─────────────────────────────────

function loadRoadMask(dataUrl) {
  const img = new Image()
  img.onload = () => {
    // Scale the mask to match the display canvas
    const cw = canvasEl.value?.width || img.width
    const ch = canvasEl.value?.height || img.height
    roadMaskW = cw
    roadMaskH = ch

    // Draw mask to a temp canvas to extract pixel data
    const tmp = document.createElement('canvas')
    tmp.width = cw
    tmp.height = ch
    const tCtx = tmp.getContext('2d')
    tCtx.imageSmoothingEnabled = true
    tCtx.imageSmoothingQuality = 'high'
    tCtx.drawImage(img, 0, 0, cw, ch)

    const imgData = tCtx.getImageData(0, 0, cw, ch)
    const px = imgData.data
    const binary = new Uint8Array(cw * ch)

    // Threshold: any pixel with brightness > 30 is a road
    for (let i = 0; i < cw * ch; i++) {
      const r = px[i * 4]
      const g = px[i * 4 + 1]
      const b = px[i * 4 + 2]
      binary[i] = (r + g + b) > 90 ? 1 : 0
    }

    roadMaskBinary = binary
    roadSettings.value = { ...roadSettings.value, hasRoadMask: true }
    nextTick(() => drawRoadOverlay())
  }
  img.src = dataUrl
}

function clearRoadMask() {
  roadMaskBinary = null
  roadMaskW = 0
  roadMaskH = 0
  roadSettings.value = { ...roadSettings.value, hasRoadMask: false }
  if (roadCanvasEl.value) {
    const ctx = roadCanvasEl.value.getContext('2d')
    ctx.clearRect(0, 0, roadCanvasEl.value.width, roadCanvasEl.value.height)
  }
}

function dilateRoadMask(binary, w, h, radius) {
  if (radius <= 1) return binary
  const out = new Uint8Array(w * h)
  const r = radius - 1
  for (let y = 0; y < h; y++) {
    for (let x = 0; x < w; x++) {
      let found = false
      for (let dy = -r; dy <= r && !found; dy++) {
        for (let dx = -r; dx <= r && !found; dx++) {
          const nx = x + dx
          const ny = y + dy
          if (nx >= 0 && nx < w && ny >= 0 && ny < h && binary[ny * w + nx]) {
            found = true
          }
        }
      }
      out[y * w + x] = found ? 1 : 0
    }
  }
  return out
}

function drawRoadOverlay() {
  if (!roadCanvasEl.value || !roadMaskBinary) return

  const cw = canvasEl.value?.width || roadMaskW
  const ch = canvasEl.value?.height || roadMaskH

  roadCanvasEl.value.width = cw
  roadCanvasEl.value.height = ch
  const ctx = roadCanvasEl.value.getContext('2d')
  ctx.clearRect(0, 0, cw, ch)

  // Parse hex color
  const hex = roadSettings.value.color
  const cr = parseInt(hex.slice(1, 3), 16)
  const cg = parseInt(hex.slice(3, 5), 16)
  const cb = parseInt(hex.slice(5, 7), 16)
  const alpha = Math.round(roadSettings.value.opacity * 255)

  // Dilate if width > 1
  const mask = dilateRoadMask(roadMaskBinary, roadMaskW, roadMaskH, roadSettings.value.width)

  const imgData = ctx.createImageData(cw, ch)
  const px = imgData.data

  for (let i = 0; i < roadMaskW * roadMaskH; i++) {
    if (mask[i]) {
      px[i * 4] = cr
      px[i * 4 + 1] = cg
      px[i * 4 + 2] = cb
      px[i * 4 + 3] = alpha
    }
  }

  ctx.putImageData(imgData, 0, 0)
}

// Re-render road overlay when settings change (no reprocessing needed)
watch(roadSettings, () => {
  if (roadSettings.value.hasRoadMask && roadMaskBinary) {
    drawRoadOverlay()
  }
}, { deep: true })

watch(image, (newImage) => {
  if (!newImage) return

  const img = new Image()
  img.onload = () => {
    originalImage = img
    imageDimensions.value = { width: img.width, height: img.height }

    const maxDim = Math.max(img.width, img.height)

    // Cap canvas dimensions to keep the browser stable
    if (maxDim > MAX_CANVAS_DIM) {
      displayScale = MAX_CANVAS_DIM / maxDim
    } else {
      displayScale = 1
    }

    const cw = Math.round(img.width * displayScale)
    const ch = Math.round(img.height * displayScale)

    if (canvasEl.value) {
      canvasEl.value.width = cw
      canvasEl.value.height = ch
      const ctx = canvasEl.value.getContext('2d')
      ctx.imageSmoothingEnabled = true
      ctx.imageSmoothingQuality = 'high'
      ctx.drawImage(img, 0, 0, cw, ch)
    }

    // Draw original at the same capped size for compare overlay
    if (originalCanvasEl.value) {
      originalCanvasEl.value.width = cw
      originalCanvasEl.value.height = ch
      const oCtx = originalCanvasEl.value.getContext('2d')
      oCtx.imageSmoothingEnabled = true
      oCtx.imageSmoothingQuality = 'high'
      oCtx.drawImage(img, 0, 0, cw, ch)
    }

    // Center and fit after the canvas is sized, then auto-segment
    nextTick(() => {
      fitToViewport()
      // If a road mask is loaded, rescale it to the new canvas size
      if (roadSettings.value.hasRoadMask && roadMaskBinary && (roadMaskW !== cw || roadMaskH !== ch)) {
        // Re-extract from the existing mask by re-triggering load isn't possible,
        // but the drawRoadOverlay will use the existing binary at the stored size.
        // Since the canvas matches, just redraw.
        drawRoadOverlay()
      }
      processImage()
    })
  }
  img.src = newImage
})
</script>
