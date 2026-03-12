<script setup lang="ts">
import { ref } from 'vue'
import ComparePanel from './components/ComparePanel.vue'
import DimensionsPanel from './components/DimensionsPanel.vue'
import ImageEditorPanel from './components/ImageEditorPanel.vue'
import TopBar from './components/TopBar.vue'

const savedMaskFile = ref<File | null>(null)
const savedMaskPreview = ref('')
const selectedSpeciesList = ref<string[]>([])

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
    <TopBar />

    <main class="main-grid">
      <ImageEditorPanel @saved="handleMaskSaved" />
      <ComparePanel
        :search-file="savedMaskFile"
        :reference-image="savedMaskPreview"
        controls-target="#compare-controls-host"
        @species-change="handleSpeciesChange"
      />
    </main>

    <DimensionsPanel :species-list="selectedSpeciesList" />
  </div>
</template>

<style scoped>
.page {
  width: 100%;
  height: 100%;
  padding: 14px;
  display: grid;
  grid-template-rows: 34px minmax(0, 0.92fr) minmax(0, 0.56fr);
  gap: 8px;
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

@media (max-width: 1200px) {
  .page {
    padding: 12px;
    grid-template-rows: 34px minmax(0, 0.9fr) minmax(0, 0.62fr);
  }

  .main-grid {
    grid-template-columns: 1fr;
    aspect-ratio: auto;
    justify-self: stretch;
  }
}
</style>
