<template>
  <button
    v-if="pinsVisible"
    class="pr-pin"
    :class="{ 'pr-pin--active': active }"
    :style="{
      left: annotation.x_pct + '%',
      top: annotation.y_pct + '%',
      background: authorColor,
    }"
    :title="annotation.author + ': ' + annotation.body"
    @click.stop="$emit('click')"
  >
    {{ index + 1 }}
    <span v-if="annotation.resolved" class="pr-pin__resolved-badge">✓</span>
    <span v-if="annotation.replies.length" class="pr-pin__reply-count">
      {{ annotation.replies.length }}
    </span>
  </button>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import type { Annotation } from '../types'
import { getAuthorColor } from '../lib/authorColor'

const props = defineProps<{
  annotation: Annotation
  index: number
  pinsVisible: boolean
  active: boolean
}>()

defineEmits<{ click: [] }>()

const authorColor = computed(() => getAuthorColor(props.annotation.author))
</script>

<style scoped>
.pr-pin {
  position: absolute;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  color: #fff;
  font-size: 11px;
  font-weight: 700;
  font-family: -apple-system, BlinkMacSystemFont, 'Inter', sans-serif;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 2px solid #fff;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.25);
  cursor: pointer;
  transform: translate(-50%, -50%);
  pointer-events: all;
  transition: transform 0.15s, box-shadow 0.15s;
  z-index: 1;
}

.pr-pin:hover,
.pr-pin--active {
  transform: translate(-50%, -50%) scale(1.2);
  box-shadow: 0 4px 14px rgba(0, 0, 0, 0.3);
}

.pr-pin__resolved-badge {
  position: absolute;
  bottom: -6px;
  left: -6px;
  width: 15px;
  height: 15px;
  border-radius: 50%;
  background: #16a34a;
  color: #fff;
  font-size: 9px;
  font-weight: 700;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 1.5px solid #fff;
}

.pr-pin__reply-count {
  position: absolute;
  top: -6px;
  right: -6px;
  min-width: 16px;
  height: 16px;
  border-radius: 8px;
  background: #1a1a1a;
  color: #fff;
  font-size: 9px;
  font-weight: 700;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0 3px;
  border: 1.5px solid #fff;
}
</style>
