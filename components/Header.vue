<template>
  <header class="flex items-center justify-between px-6 py-4 bg-slate-900 border-b border-slate-800 shrink-0 z-50">
    <div class="flex items-center gap-3">
      <div class="p-2 bg-blue-600 rounded-lg">
        <Layers class="w-6 h-6 text-white" />
      </div>
      <div>
        <h1 class="text-xl font-bold text-white tracking-tight">TerSeg</h1>
        <p class="text-xs text-slate-400 font-medium">Satellite Image Analysis &amp; K-Means Segmentation</p>
      </div>
    </div>

    <div class="flex items-center gap-4">
      <button
        @click="showDocs = true"
        class="flex items-center gap-2 px-3 py-1.5 text-sm font-medium text-slate-300 hover:text-white transition-colors"
      >
        <BookOpen class="w-4 h-4" />
        Docs
      </button>
      <button
        @click="reload"
        class="px-4 py-1.5 text-sm font-semibold bg-slate-800 text-slate-200 rounded-md hover:bg-slate-700 transition-all border border-slate-700"
      >
        Reset
      </button>
    </div>
  </header>

  <!-- Docs Modal -->
  <Teleport to="body">
    <Transition name="fade">
      <div
        v-if="showDocs"
        class="fixed inset-0 z-[100] flex items-center justify-center bg-black/60 backdrop-blur-sm"
        @mousedown.self="showDocs = false"
      >
        <div class="bg-slate-900 border border-slate-700 rounded-2xl shadow-2xl max-w-lg w-full mx-4 max-h-[80vh] flex flex-col overflow-hidden">
          <!-- Modal Header -->
          <div class="flex items-center justify-between px-6 py-4 border-b border-slate-800">
            <div class="flex items-center gap-2">
              <BookOpen class="w-5 h-5 text-blue-400" />
              <h2 class="text-lg font-bold text-white">How It Works</h2>
            </div>
            <button @click="showDocs = false" class="p-1.5 text-slate-400 hover:text-white hover:bg-slate-800 rounded-lg transition-colors">
              <X class="w-5 h-5" />
            </button>
          </div>

          <!-- Modal Body -->
          <div class="px-6 py-5 space-y-5 overflow-y-auto custom-scrollbar text-sm text-slate-300 leading-relaxed">
            <p>
              <strong class="text-white">TerSeg</strong> reduces satellite or aerial imagery into flat, uniform-color regions — ideal for game engine base maps, land cover classification, and cartographic simplification.
            </p>

            <div class="space-y-3">
              <h3 class="text-xs font-bold text-blue-400 uppercase tracking-wider">Phase 1 — Posterize</h3>
              <p>
                The image is downscaled, then box-blurred to suppress noise. <strong class="text-slate-200">K-means clustering</strong> (with k-means++ seeding) assigns every pixel to one of exactly <em>k</em> color centroids until convergence.
              </p>
            </div>

            <div class="space-y-3">
              <h3 class="text-xs font-bold text-blue-400 uppercase tracking-wider">Phase 2 — Edge Sharpen</h3>
              <p>
                A <strong class="text-slate-200">mode filter</strong> (majority vote in a local window) replaces each pixel's cluster with the most common one nearby. Multiple passes straighten wavy contour edges into crisp, blocky boundaries.
              </p>
            </div>

            <div class="space-y-3">
              <h3 class="text-xs font-bold text-blue-400 uppercase tracking-wider">Post-processing</h3>
              <p>
                Flood-fill labels connected regions, tiny regions are merged into their largest neighbour, and the result is nearest-neighbor upscaled to preserve sharp edges.
              </p>
            </div>

            <div class="p-3 bg-slate-800/60 border border-slate-700/50 rounded-xl space-y-2">
              <h3 class="text-xs font-bold text-slate-400 uppercase tracking-wider">Key Parameters</h3>
              <ul class="space-y-1 text-xs text-slate-400">
                <li><strong class="text-slate-300">Colors (k)</strong> — number of output colors</li>
                <li><strong class="text-slate-300">Blur Radius / Passes</strong> — noise reduction before clustering</li>
                <li><strong class="text-slate-300">Edge Radius / Passes</strong> — mode filter strength (more = blockier)</li>
                <li><strong class="text-slate-300">Min Region Size</strong> — regions below this are merged</li>
                <li><strong class="text-slate-300">Max Processing Size</strong> — downscale target for speed vs. detail</li>
              </ul>
            </div>

            <p class="text-xs text-slate-500">
              All processing runs in a Web Worker. Adjusting any slider automatically re-segments with a short debounce. Hold <strong class="text-slate-400">Compare</strong> in the toolbar to see the original.
            </p>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref } from 'vue'
import { Layers, BookOpen, X } from 'lucide-vue-next'

const showDocs = ref(false)

function reload() {
  window.location.reload()
}
</script>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.15s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
