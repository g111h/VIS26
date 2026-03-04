<script setup lang="ts">
interface Props {
  title?: string
  labels?: string[]
}

const props = withDefaults(defineProps<Props>(), {
  title: 'Dimensions',
  labels: () => ['sparrow', 'golden pheasant', 'red-crowned crane', 'sparrow', 'red-crowned crane']
})
</script>

<template>
  <section class="panel panel-bottom">
    <div class="panel-title">{{ props.title }}</div>
    <div class="dimension-grid">
      <div class="dimension-legend">
        <slot name="legend">
          <div class="legend-title">History</div>
          <div class="legend-item">
            <span class="legend-dot dot-tang"></span>
            Tang
          </div>
          <div class="legend-item">
            <span class="legend-dot dot-song"></span>
            Song
          </div>
          <div class="legend-item">
            <span class="legend-dot dot-yuan"></span>
            Yuan
          </div>
          <div class="legend-item">
            <span class="legend-dot dot-ming"></span>
            Ming
          </div>
        </slot>
      </div>
      <div class="ridgeline">
        <slot name="chart">
          <div class="ridge" style="--ridge-color: var(--accent-pink)"></div>
          <div class="ridge" style="--ridge-color: var(--accent-mint)"></div>
          <div class="ridge" style="--ridge-color: var(--accent-blue)"></div>
          <div class="ridge" style="--ridge-color: var(--accent-orange)"></div>
          <div class="ridge-labels">
            <span v-for="label in props.labels" :key="label">{{ label }}</span>
          </div>
        </slot>
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
  min-height: 0;
}

.panel-title {
  font-weight: 700;
  font-size: 15px;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #5f6b62;
  margin-bottom: 14px;
}

.panel-bottom {
  padding-bottom: 18px;
  overflow: hidden;
}

.dimension-grid {
  display: grid;
  grid-template-columns: 160px 1fr;
  gap: 18px;
  align-items: center;
  min-height: 0;
}

.dimension-legend {
  display: flex;
  flex-direction: column;
  gap: 10px;
  font-size: 13px;
  color: #6c746f;
}

.legend-title {
  font-weight: 700;
  color: #4d5852;
  margin-bottom: 4px;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 8px;
}

.legend-dot {
  width: 18px;
  height: 6px;
  border-radius: 999px;
  display: inline-block;
}

.dot-tang {
  background: var(--accent-pink);
}

.dot-song {
  background: var(--accent-mint);
}

.dot-yuan {
  background: var(--accent-blue);
}

.dot-ming {
  background: var(--accent-orange);
}

.ridgeline {
  position: relative;
  min-height: 170px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 8px;
}

.ridge {
  height: 28px;
  background: linear-gradient(90deg, transparent, var(--ridge-color), transparent);
  border-radius: 999px;
  opacity: 0.8;
  filter: blur(0.2px);
}

.ridge-labels {
  display: flex;
  justify-content: space-between;
  font-size: 12px;
  color: #6f7b74;
  text-transform: uppercase;
  letter-spacing: 0.08em;
}

@media (max-width: 1200px) {
  .dimension-grid {
    grid-template-columns: 1fr;
  }

  .ridgeline {
    min-height: 130px;
  }
}
</style>
