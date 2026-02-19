<template>
  <aside class="w-80 bg-slate-900 border-r border-slate-800 flex flex-col shrink-0 overflow-y-auto custom-scrollbar">
    <div class="p-6 space-y-8">
      <!-- Image Info -->
      <div v-if="imageDimensions" class="p-3 bg-slate-800/50 border border-slate-700/50 rounded-xl space-y-1">
        <div class="text-[10px] font-bold text-slate-500 uppercase tracking-wider">Source Image</div>
        <div class="text-xs text-slate-300 font-mono">
          {{ imageDimensions.width }} × {{ imageDimensions.height }}px
          <span class="text-slate-500">({{ Math.round(imageDimensions.width * imageDimensions.height / 1e6 * 10) / 10 }}MP)</span>
        </div>
        <div v-if="imageDimensions.width > 4096 || imageDimensions.height > 4096" class="text-[10px] text-amber-400/80">
          Large image — consider lowering max processing size
        </div>
      </div>

      <!-- Algorithm Settings -->
      <div class="space-y-6">
        <div class="flex items-center gap-2 text-slate-400 font-bold text-xs uppercase tracking-wider">
          <Sliders class="w-4 h-4" />
          Segmentation Params
        </div>

        <div class="space-y-4">
          <div class="space-y-2">
            <div class="flex justify-between text-xs font-medium">
              <label class="text-slate-300">Number of Colors (k)</label>
              <span class="text-blue-400">{{ params.numColors }}</span>
            </div>
            <input
              type="range" :min="2" :max="32" :step="1"
              :value="params.numColors"
              @input="updateParam('numColors', parseInt($event.target.value))"
              class="w-full h-1.5 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-blue-500"
            />
            <p class="text-[10px] text-slate-500">Exact number of output colors via k-means clustering.</p>
          </div>

          <div class="space-y-2">
            <div class="flex justify-between text-xs font-medium">
              <label class="text-slate-300">Blur Radius</label>
              <span class="text-blue-400">{{ params.blurRadius }}px</span>
            </div>
            <input
              type="range" :min="0" :max="20" :step="1"
              :value="params.blurRadius"
              @input="updateParam('blurRadius', parseInt($event.target.value))"
              class="w-full h-1.5 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-blue-500"
            />
            <p class="text-[10px] text-slate-500">Box blur kernel radius for noise reduction before clustering.</p>
          </div>

          <div class="space-y-2">
            <div class="flex justify-between text-xs font-medium">
              <label class="text-slate-300">Blur Passes</label>
              <span class="text-blue-400">{{ params.blurPasses }}</span>
            </div>
            <input
              type="range" :min="1" :max="6" :step="1"
              :value="params.blurPasses"
              @input="updateParam('blurPasses', parseInt($event.target.value))"
              class="w-full h-1.5 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-blue-500"
            />
            <p class="text-[10px] text-slate-500">Number of blur iterations. 3 passes ≈ Gaussian blur.</p>
          </div>

          <div class="space-y-2">
            <div class="flex justify-between text-xs font-medium">
              <label class="text-slate-300">Edge Sharpening Radius</label>
              <span class="text-blue-400">{{ params.modeRadius }}px</span>
            </div>
            <input
              type="range" :min="1" :max="8" :step="1"
              :value="params.modeRadius"
              @input="updateParam('modeRadius', parseInt($event.target.value))"
              class="w-full h-1.5 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-blue-500"
            />
            <p class="text-[10px] text-slate-500">Mode filter window radius. Larger = blockier edges.</p>
          </div>

          <div class="space-y-2">
            <div class="flex justify-between text-xs font-medium">
              <label class="text-slate-300">Edge Sharpening Passes</label>
              <span class="text-blue-400">{{ params.modePasses }}</span>
            </div>
            <input
              type="range" :min="1" :max="8" :step="1"
              :value="params.modePasses"
              @input="updateParam('modePasses', parseInt($event.target.value))"
              class="w-full h-1.5 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-blue-500"
            />
            <p class="text-[10px] text-slate-500">More passes = crisper, more uniform boundaries.</p>
          </div>

          <div class="space-y-2">
            <div class="flex justify-between text-xs font-medium">
              <label class="text-slate-300">Min Region Size</label>
              <span class="text-blue-400">{{ params.minRegionSize }}px</span>
            </div>
            <input
              type="range" :min="1" :max="500" :step="5"
              :value="params.minRegionSize"
              @input="updateParam('minRegionSize', parseInt($event.target.value))"
              class="w-full h-1.5 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-blue-500"
            />
            <p class="text-[10px] text-slate-500">Regions smaller than this are merged into neighbors.</p>
          </div>

          <div class="space-y-2">
            <div class="flex justify-between text-xs font-medium">
              <label class="text-slate-300">Max Processing Size</label>
              <span class="text-blue-400">{{ params.maxSize }}px</span>
            </div>
            <select
              :value="params.maxSize"
              @change="updateParam('maxSize', parseInt($event.target.value))"
              class="w-full bg-slate-800 border border-slate-700 text-slate-300 text-xs rounded-lg p-2 focus:ring-blue-500 focus:border-blue-500"
            >
              <option :value="256">256px (Fast Preview)</option>
              <option :value="512">512px (Balanced)</option>
              <option :value="768">768px (Higher Detail)</option>
              <option :value="1024">1024px (High Quality)</option>
            </select>
            <p class="text-[10px] text-slate-500">Image is downscaled to this size for processing, then upscaled with nearest-neighbor.</p>
          </div>
        </div>
      </div>

      <!-- Road Overlay Settings -->
      <div class="space-y-6">
        <div class="flex items-center gap-2 text-slate-400 font-bold text-xs uppercase tracking-wider">
          <Route class="w-4 h-4" />
          Road Overlay
        </div>

        <div class="space-y-4">
          <!-- Upload Road Mask -->
          <div class="space-y-2">
            <input type="file" class="hidden" accept="image/png" ref="roadFileInput" @change="handleRoadUpload" />
            <div class="flex items-center gap-2">
              <button
                @click="$refs.roadFileInput?.click()"
                class="flex-1 flex items-center justify-center gap-2 px-3 py-2 bg-slate-800 hover:bg-slate-700 border border-slate-700 rounded-lg text-xs font-medium text-slate-300 transition-colors"
              >
                <Upload class="w-3.5 h-3.5" />
                {{ roadSettings.hasRoadMask ? 'Replace Mask' : 'Upload Road Mask' }}
              </button>
              <button
                v-if="roadSettings.hasRoadMask"
                @click="clearRoadMask"
                class="p-2 bg-slate-800 hover:bg-red-900/40 border border-slate-700 rounded-lg text-slate-400 hover:text-red-400 transition-colors"
                title="Remove road overlay"
              >
                <X class="w-3.5 h-3.5" />
              </button>
            </div>
            <p class="text-[10px] text-slate-500">16-bit PNG mask: white roads on black background.</p>
          </div>

          <!-- Road Opacity -->
          <div v-if="roadSettings.hasRoadMask" class="space-y-2">
            <div class="flex justify-between text-xs font-medium">
              <label class="text-slate-300">Road Opacity</label>
              <span class="text-blue-400">{{ Math.round(roadSettings.opacity * 100) }}%</span>
            </div>
            <input
              type="range" :min="0" :max="1" :step="0.05"
              :value="roadSettings.opacity"
              @input="updateRoadSetting('opacity', parseFloat($event.target.value))"
              class="w-full h-1.5 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-blue-500"
            />
          </div>

          <!-- Road Color -->
          <div v-if="roadSettings.hasRoadMask" class="space-y-2">
            <div class="text-xs font-medium text-slate-300">Road Color</div>
            <div class="flex gap-2 flex-wrap">
              <button
                v-for="c in roadColorPresets" :key="c.value"
                @click="updateRoadSetting('color', c.value)"
                class="w-7 h-7 rounded-lg border-2 transition-all"
                :class="roadSettings.color === c.value ? 'border-blue-400 scale-110' : 'border-slate-600 hover:border-slate-400'"
                :style="{ backgroundColor: c.value }"
                :title="c.label"
              />
            </div>
          </div>

          <!-- Road Width -->
          <div v-if="roadSettings.hasRoadMask" class="space-y-2">
            <div class="flex justify-between text-xs font-medium">
              <label class="text-slate-300">Road Width</label>
              <span class="text-blue-400">{{ roadSettings.width }}px</span>
            </div>
            <input
              type="range" :min="1" :max="8" :step="1"
              :value="roadSettings.width"
              @input="updateRoadSetting('width', parseInt($event.target.value))"
              class="w-full h-1.5 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-blue-500"
            />
            <p class="text-[10px] text-slate-500">Dilate road lines for visibility. 1 = original mask size.</p>
          </div>
        </div>
      </div>

      <!-- Processing Status -->
      <div
        v-if="isProcessing"
        class="flex items-center justify-center gap-2 py-2 px-4 bg-slate-800/60 border border-slate-700/50 rounded-xl text-xs text-slate-400"
      >
        <div class="w-3.5 h-3.5 border-2 border-blue-400/30 border-t-blue-400 rounded-full animate-spin"></div>
        Segmenting...
      </div>

      <!-- Tooltip Info -->
      <div class="pt-4 border-t border-slate-800 space-y-3">
        <div class="flex gap-2 text-slate-500">
          <Info class="w-4 h-4 shrink-0 mt-0.5" />
          <p class="text-[10px] leading-relaxed">
            <strong class="text-slate-400">Phase 1 (Posterize):</strong> Blur + k-means reduces to exactly <em>k</em> colors.
            <strong class="text-slate-400">Phase 2 (Sharpen):</strong> Mode filter snaps wavy contours into crisp blocky edges.
          </p>
        </div>
      </div>
    </div>
  </aside>
</template>

<script setup>
import { ref } from 'vue'
import { Sliders, Info, Route, Upload, X } from 'lucide-vue-next'

const props = defineProps({
  params: {
    type: Object,
    required: true
  },
  isProcessing: {
    type: Boolean,
    default: false
  },
  imageDimensions: {
    type: Object,
    default: null
  },
  roadSettings: {
    type: Object,
    required: true
  }
})

const emit = defineEmits(['update:params', 'update:roadSettings', 'road-mask-file', 'clear-road-mask'])

const roadFileInput = ref(null)

const roadColorPresets = [
  { label: 'Asphalt', value: '#4a4a4a' },
  { label: 'White', value: '#ffffff' },
  { label: 'Yellow', value: '#facc15' },
  { label: 'Orange', value: '#f97316' },
  { label: 'Red', value: '#ef4444' },
  { label: 'Cyan', value: '#22d3ee' },
  { label: 'Lime', value: '#84cc16' }
]

function updateParam(key, value) {
  emit('update:params', { ...props.params, [key]: value })
}

function updateRoadSetting(key, value) {
  emit('update:roadSettings', { ...props.roadSettings, [key]: value })
}

function handleRoadUpload(e) {
  const file = e.target?.files?.[0]
  if (!file) return
  const reader = new FileReader()
  reader.onload = (ev) => {
    emit('road-mask-file', ev.target?.result)
  }
  reader.readAsDataURL(file)
  e.target.value = ''
}

function clearRoadMask() {
  emit('clear-road-mask')
}
</script>
