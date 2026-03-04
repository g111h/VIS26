<script setup lang="ts">
interface Props {
  title?: string
  actionLabel?: string
}

const props = withDefaults(defineProps<Props>(), {
  title: 'P2P-AVA',
  actionLabel: '芙 蓉 锦 鸡 图'
})
</script>

<template>
  <header class="topbar">
    <div class="brand">
      <slot name="title">{{ props.title }}</slot>
    </div>
    <slot name="action">
      <button class="pill-btn" type="button">{{ props.actionLabel }}</button>
    </slot>
  </header>
</template>

<style scoped>
.topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  min-width: 0;
  height: 100%;
}

.brand {
  font-family: 'DM Serif Display', serif;
  font-size: 18px;
  letter-spacing: 1px;
  line-height: 1;
}

.pill-btn {
  border: 1px solid var(--panel-border);
  background: #ffffff;
  border-radius: 999px;
  padding: 5px 12px;
  font-weight: 600;
  font-size: 12px;
  line-height: 1.2;
  color: var(--accent-ink);
  cursor: pointer;
  box-shadow: var(--shadow-soft);
}

@media (max-width: 720px) {
  .topbar {
    flex-direction: column;
    align-items: flex-start;
    gap: 10px;
  }
}
</style>
