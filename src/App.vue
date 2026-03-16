<script setup lang="ts">
import { ref } from 'vue'
import ComparePanel from './components/ComparePanel.vue'
import DimensionsPanel from './components/DimensionsPanel.vue'
import ImageEditorPanel from './components/ImageEditorPanel.vue'
import TopBar from './components/TopBar.vue'

const savedMaskFile = ref<File | null>(null)
const savedMaskPreview = ref('')
const selectedSpeciesList = ref<string[]>([])
const selectedPainting = ref<'furongjinjitu' | 'zhuquetu'>('furongjinjitu')
const dimensionLayoutMode = ref<'overlay' | 'stacked'>('overlay')

function handleMaskSaved(payload: { maskUrl: string; file: File }) {
  savedMaskFile.value = payload.file
  savedMaskPreview.value = payload.maskUrl
}

function handleSpeciesChange(speciesList: string[]) {
  selectedSpeciesList.value = speciesList
}
</script>

<template>
  <div class="page">
    <TopBar>
      <template #action>
        <div class="painting-select-wrap">
          <div class="layout-mode-switch" role="group" aria-label="dimension layout mode">
            <button
              class="layout-mode-btn"
              :class="{ 'is-active': dimensionLayoutMode === 'overlay' }"
              type="button"
              @click="dimensionLayoutMode = 'overlay'"
            >
              重叠
            </button>
            <button
              class="layout-mode-btn"
              :class="{ 'is-active': dimensionLayoutMode === 'stacked' }"
              type="button"
              @click="dimensionLayoutMode = 'stacked'"
            >
              堆叠
            </button>
          </div>
          <select v-model="selectedPainting" class="painting-select" aria-label="选择画作">
            <option value="furongjinjitu">芙蓉锦鸡图</option>
            <option value="zhuquetu">竹雀图</option>
          </select>
        </div>
      </template>
    </TopBar>

    <main class="main-grid">
      <ImageEditorPanel
        :preset-image="selectedPainting"
        @saved="handleMaskSaved"
      />
      <ComparePanel
        :search-file="savedMaskFile"
        :reference-image="savedMaskPreview"
        controls-target="#compare-controls-host"
        @species-change="handleSpeciesChange"
      />
    </main>

    <DimensionsPanel :species-list="selectedSpeciesList" :layout-mode="dimensionLayoutMode" />
  </div>
</template>

<style scoped>
.page {
  width: 100%;
  height: 100%;
  padding: 10px 14px 14px;
  display: grid;
  grid-template-rows: 30px minmax(0, 0.92fr) minmax(0, 0.56fr);
  gap: 6px;
  box-sizing: border-box;
  overflow: hidden;
}

.main-grid {
  display: grid;
  grid-template-columns: minmax(0, 46fr) minmax(0, 54fr);
  aspect-ratio: 6 / 5;
  width: 100%;
  max-height: 100%;
  min-height: 0;
  gap: 8px;
  justify-self: center;
}

.painting-select-wrap {
  display: flex;
  align-items: center;
  gap: 8px;
}

.layout-mode-switch {
  display: inline-flex;
  border: 1px solid var(--panel-border);
  border-radius: 999px;
  background: #ffffff;
  box-shadow: var(--shadow-soft);
  overflow: hidden;
}

.layout-mode-btn {
  border: none;
  background: transparent;
  color: #52635c;
  font-weight: 700;
  font-size: 12px;
  line-height: 1.2;
  padding: 6px 10px;
  cursor: pointer;
}

.layout-mode-btn.is-active {
  background: #3d4f4a;
  color: #ffffff;
}

.painting-select {
  border: 1px solid var(--panel-border);
  background: #ffffff;
  border-radius: 999px;
  padding: 6px 28px 6px 12px;
  font-weight: 600;
  font-size: 12px;
  line-height: 1.2;
  color: var(--accent-ink);
  box-shadow: var(--shadow-soft);
  cursor: pointer;
}

@media (max-width: 1200px) {
  .page {
    padding: 9px 12px 12px;
    grid-template-rows: 28px minmax(0, 0.9fr) minmax(0, 0.62fr);
    gap: 6px;
  }

  .main-grid {
    grid-template-columns: 1fr;
    aspect-ratio: auto;
    justify-self: stretch;
  }
}
</style>
