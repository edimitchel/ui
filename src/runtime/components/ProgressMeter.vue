<!-- eslint-disable vue/block-tag-newline -->
<script lang="ts">
import type { ProgressRootProps, ProgressRootEmits } from 'reka-ui'
import type { AppConfig } from '@nuxt/schema'
import theme from '#build/ui/progress'
import type { ComponentConfig } from '../types/tv'
import { injectProgress } from './Progress.vue'

type ProgressMeter = ComponentConfig<typeof theme, AppConfig, 'progress-meter'>

export interface ProgressMeterProps extends Pick<ProgressRootProps, 'getValueLabel' | 'getValueText' | 'modelValue'> {
  as?: any
  color?: ProgressMeter['variants']['color']
  class?: any
  ui?: ProgressMeter['slots']
}

export interface ProgressMeterEmits extends ProgressRootEmits {
}

</script>

<script setup lang="ts">
import { computed, onMounted } from 'vue'
import { Primitive, ProgressIndicator, ProgressRoot } from 'reka-ui'
import { useAppConfig } from '#imports'
import { tv } from '../utils/tv'
import { useLocale } from '../composables/useLocale'

const { dir } = useLocale()

const props = withDefaults(defineProps<ProgressMeterProps>(), {
  inverted: false,
  modelValue: null,
  orientation: 'horizontal'
})

const appConfig = useAppConfig() as ProgressMeter['AppConfig']

const { max, updateGroupValue } = injectProgress()

const percent = computed(() => {
  switch (true) {
    case props.modelValue! < 0: return 0
    case props.modelValue! > max.value: return max.value
    default: return Math.round((props.modelValue! / max.value)) * 100
  }
})

const indicatorStyle = computed(() => {
  if (percent.value === undefined) {
    return
  }

  if (props.orientation === 'vertical') {
    return {
      transform: `translateY(${props.inverted ? '' : '-'}${100 - percent.value}%)`
    }
  } else {
    if (dir.value === 'rtl') {
      return {
        transform: `translateX(${props.inverted ? '-' : ''}${100 - percent.value}%)`
      }
    } else {
      return {
        transform: `translateX(${props.inverted ? '' : '-'}${100 - percent.value}%)`
      }
    }
  }
})

const ui = computed(() => tv({ extend: tv(theme), ...(appConfig.ui?.['progress-meter'] || {}) })({
  color: props.color
}))

// Use inject to update parent's total value for group calculation
onMounted(() => {
  if (typeof props.modelValue === 'number') {
    updateGroupValue(props.modelValue)
  }
})
</script>

<template>
  <div :class="ui.base({ class: props.ui?.base })" />
  <!-- <ProgressIndicator :class="ui.indicator({ class: props.ui?.indicator })" :style="indicatorStyle" /> -->
</template>
