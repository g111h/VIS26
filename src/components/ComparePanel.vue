<script setup lang="ts">
interface WeightItem {
  label: string
  value: number
  tone?: 'pink' | 'mint' | 'blue'
}

interface Props {
  title?: string
  promptPlaceholder?: string
  weights?: WeightItem[]
  compareImage?: string
  referenceImage?: string
  galleryImages?: string[]
}

const props = withDefaults(defineProps<Props>(), {
  title: 'Compare',
  promptPlaceholder: 'Describe the style, palette, and mood',
  weights: () => [
    { label: 'Texture', value: 60, tone: 'pink' },
    { label: 'Structure', value: 40, tone: 'mint' },
    { label: 'Color', value: 60, tone: 'blue' }
  ],
  compareImage:
    'https://images.unsplash.com/photo-1500530855697-b586d89ba3ee?auto=format&fit=crop&w=900&q=80',
  referenceImage:
    'https://images.unsplash.com/photo-1500530855697-b586d89ba3ee?auto=format&fit=crop&w=800&q=80',
  galleryImages: () => [
    'https://images.unsplash.com/photo-1474511320723-9a56873867b5?auto=format&fit=crop&w=300&q=80',
    'https://images.unsplash.com/photo-1452570053594-1b985d6ea890?auto=format&fit=crop&w=300&q=80',
    'https://images.unsplash.com/photo-1465156799763-2c087c332922?auto=format&fit=crop&w=300&q=80',
    'https://images.unsplash.com/photo-1470104240373-bc1812eddc9f?auto=format&fit=crop&w=300&q=80',
    'https://images.unsplash.com/photo-1480044965905-02098d419e96?auto=format&fit=crop&w=300&q=80',
    'https://images.unsplash.com/photo-1444464666168-49d633b86797?auto=format&fit=crop&w=300&q=80'
  ]
})
</script>

<template>
  <section class="panel panel-compare">
    <div class="compare-grid">
      <div class="compare-column compare-column-controls">
        <div class="prompt-card">
          <div class="prompt-header">
            <span class="prompt-search-icon" aria-hidden="true">🔍</span>
            <span class="prompt-title">Prompt</span>
          </div>
          <div class="prompt-input">
            <slot name="prompt">
              <textarea
                :placeholder="props.promptPlaceholder"
                aria-label="prompt"
                rows="4"
              ></textarea>
            </slot>
          </div>
          <div class="weights">
            <div class="weights-title">Field weights</div>
            <slot name="fieldweights">
              <div class="weights-list">
                <div
                  v-for="weight in props.weights"
                  :key="weight.label"
                  class="weight-row"
                >
                  <span>{{ weight.label }}</span>
                  <div class="weight-track">
                    <span
                      class="weight-fill"
                      :class="{
                        'is-pink': weight.tone === 'pink',
                        'is-mint': weight.tone === 'mint',
                        'is-blue': weight.tone === 'blue'
                      }"
                      :style="{ width: `${weight.value}%` }"
                    ></span>
                  </div>
                  <span class="weight-value">{{ weight.value }}%</span>
                </div>
              </div>
            </slot>
          </div>
        </div>
      </div>

      <div class="compare-column">
        <slot name="compare">
          <div class="orbit">
            <div class="orbit-core">
              <img :src="props.compareImage" alt="center" />
            </div>
            <div class="orbit-ring ring-1"></div>
            <div class="orbit-ring ring-2"></div>
            <div class="orbit-ring ring-3"></div>
            <div class="orbit-node n1"></div>
            <div class="orbit-node n2"></div>
            <div class="orbit-node n3"></div>
            <div class="orbit-node n4"></div>
            <div class="orbit-node n5"></div>
            <div class="orbit-node n6"></div>
            <div class="orbit-node n7"></div>
            <div class="orbit-node n8"></div>
          </div>
        </slot>
      </div>

      <div class="compare-column compare-column-media">
        <div class="panel-block">
          <div class="panel-subtitle">Image</div>
          <slot name="image">
            <div class="hero-image">
              <img :src="props.referenceImage" alt="reference" />
            </div>
          </slot>
        </div>
        <div class="panel-block gallery-block">
          <div class="panel-subtitle">Gallery</div>
          <slot name="gallery">
            <div class="gallery-grid">
              <img
                v-for="(image, index) in props.galleryImages"
                :key="`${image}-${index}`"
                :src="image"
                :alt="`thumb ${index + 1}`"
              />
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
  grid-template-columns: 0.72fr 1.18fr 0.9fr;
  gap: 16px;
  align-items: stretch;
  flex: 1;
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
}

.weights-list {
  min-height: 0;
  flex: 1;
}

.weight-row {
  display: grid;
  grid-template-columns: 90px 1fr 40px;
  align-items: center;
  gap: 12px;
  font-size: 13px;
  margin-top: 8px;
}

.weight-track {
  height: 6px;
  background: #e5ede5;
  border-radius: 999px;
  overflow: hidden;
}

.weight-fill {
  display: block;
  height: 100%;
  border-radius: 999px;
}

.weight-fill.is-pink {
  background: var(--accent-pink);
}

.weight-fill.is-mint {
  background: var(--accent-mint);
}

.weight-fill.is-blue {
  background: var(--accent-blue);
}

.weight-value {
  color: #6d7b73;
  font-weight: 600;
}

.orbit {
  position: relative;
  flex: 1;
  border-radius: 18px;
  border: 1px dashed #d7ded4;
  background: radial-gradient(circle at center, #f4f7f2 0%, #fdfdfb 50%, #f4f7f2 100%);
  min-height: 0;
  overflow: hidden;
}

.orbit-core {
  position: absolute;
  width: 110px;
  height: 110px;
  border-radius: 50%;
  overflow: hidden;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  border: 6px solid #ffffff;
  box-shadow: 0 8px 20px rgba(50, 60, 55, 0.15);
}

.orbit-ring {
  position: absolute;
  border: 1px solid #e0e6dd;
  border-radius: 50%;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
}

.ring-1 {
  width: 180px;
  height: 180px;
}

.ring-2 {
  width: 260px;
  height: 260px;
}

.ring-3 {
  width: 340px;
  height: 340px;
}

.orbit-node {
  position: absolute;
  width: 52px;
  height: 52px;
  border-radius: 50%;
  border: 4px solid rgba(255, 255, 255, 0.9);
  background: linear-gradient(135deg, #f8d7d0, #fdebd8);
  box-shadow: 0 10px 20px rgba(60, 70, 65, 0.12);
}

.orbit-node::after {
  content: '';
  position: absolute;
  inset: 10px;
  border-radius: 50%;
  background: linear-gradient(120deg, #d4e8ff, #f2f7ff);
}

.orbit-node.n1 {
  left: 18%;
  top: 18%;
}

.orbit-node.n2 {
  left: 65%;
  top: 12%;
}

.orbit-node.n3 {
  left: 80%;
  top: 45%;
}

.orbit-node.n4 {
  left: 65%;
  top: 70%;
}

.orbit-node.n5 {
  left: 30%;
  top: 72%;
}

.orbit-node.n6 {
  left: 12%;
  top: 50%;
}

.orbit-node.n7 {
  left: 45%;
  top: 8%;
}

.orbit-node.n8 {
  left: 48%;
  top: 78%;
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
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: #6a756c;
}

.hero-image {
  border-radius: 14px;
  overflow: hidden;
  height: 160px;
}

.hero-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
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

.gallery-grid img {
  border-radius: 10px;
  height: 80px;
  object-fit: cover;
}

@media (max-width: 1200px) {
  .compare-grid {
    grid-template-columns: 1fr;
  }

  .orbit {
    min-height: 220px;
  }
}

@media (max-width: 720px) {
  .gallery-grid {
    grid-template-columns: repeat(3, 1fr);
  }
}
</style>
