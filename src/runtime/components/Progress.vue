<!-- eslint-disable vue/block-tag-newline -->
<script lang="ts">
import type { ProgressRootProps, ProgressRootEmits } from 'reka-ui'
import type { AppConfig } from '@nuxt/schema'
import theme from '#build/ui/progress'
import type { ComponentConfig } from '../types/tv'

export type Progress = ComponentConfig<typeof theme, AppConfig, 'progress'>

export interface ProgressProps extends Pick<ProgressRootProps, 'getValueLabel' | 'getValueText' | 'modelValue'> {
  /**
   * The element or component this component should render as.
   * @defaultValue 'div'
   */
  as?: any
  /** The maximum progress value. */
  max?: number | Array<any>
  /** Display the current progress value. */
  status?: boolean
  /** Whether the progress is visually inverted. */
  inverted?: boolean
  /**
   * @defaultValue 'md'
   */
  size?: Progress['variants']['size']
  /**
   * @defaultValue 'primary'
   */
  color?: Progress['variants']['color']
  /**
   * The orientation of the progress bar.
   * @defaultValue 'horizontal'
   */
  orientation?: Progress['variants']['orientation']
  /**
   * The animation of the progress bar.
   * @defaultValue 'carousel'
   */
  animation?: Progress['variants']['animation']
  class?: any
  ui?: Progress['slots']
}

export interface ProgressEmits extends ProgressRootEmits {}

export type ProgressSlots = {
  status(props: { percent?: number }): any
  default(): any
} & {
  [key: string]: (props: { step: number }) => any
}

export interface ProvideProgressContext {
  updateGroupValue: (value: number) => void
  max: ComputedRef<number>
}

export const [injectProgress, provideProgress] = createContext<ProvideProgressContext>('progress')

</script>

<script setup lang="ts">
import { computed, ref, onMounted, type ComputedRef } from 'vue'
import { Primitive, ProgressRoot, ProgressIndicator, useForwardPropsEmits, createContext } from 'reka-ui'
import { reactivePick } from '@vueuse/core'
import { useAppConfig } from '#imports'
import { useLocale } from '../composables/useLocale'
import { tv } from '../utils/tv'

const props = withDefaults(defineProps<ProgressProps>(), {
  inverted: false,
  modelValue: null,
  orientation: 'horizontal'
})
const emits = defineEmits<ProgressEmits>()
const slots = defineSlots<ProgressSlots>()

const { dir } = useLocale()
const appConfig = useAppConfig() as Progress['AppConfig']

const rootProps = useForwardPropsEmits(reactivePick(props, 'getValueLabel', 'getValueText', 'modelValue'), emits)

const isGrouped = computed(() => slots.default())

const totalGroupValue = ref(0)

if (isGrouped.value) {
// Provide a function for child components to update the total value
  provideProgress({
    updateGroupValue: (value: number) => {
      totalGroupValue.value += value
    },
    max: computed(() => Number(props.max))
  })
}

// Reset total value on mount if group is active
onMounted(() => {
  if (isGrouped.value) {
    totalGroupValue.value = 0
  }
})

const isIndeterminate = computed(() => !isGrouped.value && rootProps.value.modelValue === null)
const hasSteps = computed(() => Array.isArray(props.max))

const realMax = computed(() => {
  if (isIndeterminate.value || (!props.max && !isGrouped.value)) {
    return undefined
  }
  if (isGrouped.value && !props.max) {
    return totalGroupValue.value || 100
  }
  if (Array.isArray(props.max)) {
    return props.max.length - 1
  }

  return Number(props.max)
})

const percent = computed(() => {
  if (isIndeterminate.value) {
    return undefined
  }

  switch (true) {
    case rootProps.value.modelValue! < 0: return 0
    case rootProps.value.modelValue! > (realMax.value ?? 100): return 100
    default: return Math.round((rootProps.value.modelValue! / (realMax.value ?? 100)) * 100)
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

const statusStyle = computed(() => {
  const value = `${Math.max(percent.value ?? 0, 0)}%`
  return props.orientation === 'vertical' ? { height: value } : { width: value }
})

function isActive(index: number) {
  return index === Number(props.modelValue)
}

function isFirst(index: number) {
  return index === 0
}

function isLast(index: number) {
  return index === realMax.value
}

function stepVariant(index: number | string) {
  index = Number(index)

  if (isActive(index) && !isFirst(index)) {
    return 'active'
  }

  if (isFirst(index) && isActive(index)) {
    return 'first'
  }

  if (isLast(index) && isActive(index)) {
    return 'last'
  }

  return 'other'
}

const ui = computed(() => tv({ extend: tv(theme), ...(appConfig.ui?.progress || {}) })({
  animation: props.animation,
  size: props.size,
  color: props.color,
  orientation: props.orientation,
  inverted: props.inverted
}))
</script>

<template>
  <Primitive :as="as" :data-orientation="orientation" :class="ui.root({ class: [props.ui?.root, props.class] })">
    <div v-if="!isIndeterminate && (status || !!slots.status)" :class="ui.status({ class: props.ui?.status })" :style="statusStyle">
      <slot name="status" :percent="percent">
        {{ percent }}%
      </slot>
    </div>

    <ProgressRoot v-bind="rootProps" :max="realMax" :class="ui.base({ class: props.ui?.base })" style="transform: translateZ(0)">
      <slot>
        <ProgressIndicator :class="ui.indicator({ class: props.ui?.indicator })" :style="indicatorStyle" />
      </slot>
    </ProgressRoot>

    <div v-if="hasSteps" :class="ui.steps({ class: props.ui?.steps })">
      <div v-for="(step, index) in max" :key="index" :class="ui.step({ class: props.ui?.step, step: stepVariant(index) })">
        <slot :name="`step-${index}`" :step="step">
          {{ step }}
        </slot>
      </div>
    </div>
    <slot />
  </Primitive>
</template>
