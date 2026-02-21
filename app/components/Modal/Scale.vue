<script setup lang="ts">
const props = defineProps<{
  isOpen: boolean
  name: string
  type: ScaleType
  count: number
  options: { name: string; value: number }[]
  choices?: { index: number; value: number | null }[]
}>()
const emit = defineEmits<{
  (event: 'close'): void
  (event: 'calculate', data: { index: number; value: number | null }[]): void
}>()

const { proxy: gaProxy } = useScriptGoogleAnalytics()

const choices = ref<{ index: number; value: number | null }[]>(props.choices ?? new Array(props.count).fill(0).map((_, index) => ({ index: index + 1, value: null })))

watch(
  () => props.choices,
  (value) => {
    if (value) choices.value = value
  }
)

const choicesSlides = computed(() => {
  const slides = []
  const groups = []
  const groupPerSlide = props.type === 'binary' ? 6 : 4

  for (let i = 0; i < choices.value.length; i += 5) {
    groups.push(choices.value.slice(i, Math.min(i + 5, choices.value.length)))
  }
  for (let i = 0; i < groups.length; i += groupPerSlide) {
    slides.push(groups.slice(i, Math.min(i + groupPerSlide, groups.length)))
  }
  return slides
})

const currentSlide = ref(0)

const totalSlides = computed(() => choicesSlides.value.length + 1)
const lastQuestionSlideIdx = computed(() => choicesSlides.value.length - 1)

const isLastSlide = computed(() => currentSlide.value === lastQuestionSlideIdx.value)

const [sliderRef, slider] = useKeenSlider({
  initial: 0,
  drag: false,
  slideChanged(s) {
    currentSlide.value = s.track.details.rel
  },
})

function goTo(idx: number) {
  slider.value?.moveToIdx(idx)
}
function goNext() {
  slider.value?.next()
}
function goPrev() {
  slider.value?.prev()
}

const result = ref<{ name: string; value: number }[]>()
const isLoading = ref(false)
const currentChoiceIndex = ref(0)
const currentChoiceValue = computed(() => choices.value[currentChoiceIndex.value]!.value)
const invalidChoiceIndex = ref<number | null>(null)
const minLimit = computed(() => props.options.reduce((min, { value }) => (value < min ? value : min), 9999))
const maxLimit = computed(() => props.options.reduce((max, { value }) => (value > max ? value : max), 0))

watch(choices, () => {
  result.value = undefined
})

watch(currentChoiceIndex, (value) => {
  const groupPerSlide = props.type === 'binary' ? 6 : 4
  goTo(Math.floor(value / (5 * groupPerSlide)))
})

function onInput(index: number, value: number) {
  if (!(value >= minLimit.value && value <= maxLimit.value)) return
  choices.value[index]!.value = value
  currentChoiceIndex.value = index
}

function checkScaleItemValidity({ minLimit, maxLimit }: { minLimit: number; maxLimit: number }, data: { index: number; value: number }[]) {
  invalidChoiceIndex.value = null

  for (const { index, value } of data) {
    if (value === null || !(value >= minLimit && value <= maxLimit)) {
      invalidChoiceIndex.value = index - 1
      break
    }
  }

  if (invalidChoiceIndex.value === null) return true

  const groupPerSlide = props.type === 'binary' ? 6 : 4
  const targetSlide = Math.floor(invalidChoiceIndex.value / (5 * groupPerSlide))
  goTo(targetSlide)
  currentChoiceIndex.value = invalidChoiceIndex.value
  result.value = undefined
  return false
}

const { arrowLeft, arrowRight, arrowUp, arrowDown, t, f, numpad0, numpad1, numpad2, numpad3, numpad4, numpad5, digit0, digit1, digit2, digit3, digit4, digit5 } = useMagicKeys()

watchArray(
  [arrowLeft, arrowRight, arrowUp, arrowDown, t, f, numpad0, numpad1, numpad2, numpad3, numpad4, numpad5, digit0, digit1, digit2, digit3, digit4, digit5],
  ([left, right, up, down, t, f, numpad0, numpad1, numpad2, numpad3, numpad4, numpad5, digit0, digit1, digit2, digit3, digit4, digit5]) => {
    if (left) {
      const dir = props.type === 'binary' ? -1 : 1
      const next = currentChoiceValue.value !== null ? Math.max(currentChoiceValue.value - 1 * dir, minLimit.value) : props.type === 'binary' ? maxLimit.value : minLimit.value
      onInput(currentChoiceIndex.value, next)
    } else if (right) {
      const dir = props.type === 'binary' ? -1 : 1
      const next = currentChoiceValue.value !== null ? Math.min(currentChoiceValue.value + 1 * dir, maxLimit.value) : props.type === 'binary' ? maxLimit.value : minLimit.value
      onInput(currentChoiceIndex.value, next)
    } else if (up) {
      currentChoiceIndex.value -= currentChoiceIndex.value > 0 ? 1 : 0
    } else if (down) {
      currentChoiceIndex.value += currentChoiceIndex.value < choices.value.length - 1 ? 1 : 0
    } else if (f || numpad0 || digit0) {
      onInput(currentChoiceIndex.value, 0)
    } else if (t || numpad1 || digit1) {
      onInput(currentChoiceIndex.value, 1)
    } else if (numpad2 || digit2) {
      onInput(currentChoiceIndex.value, 2)
    } else if (numpad3 || digit3) {
      onInput(currentChoiceIndex.value, 3)
    } else if (numpad4 || digit4) {
      onInput(currentChoiceIndex.value, 4)
    } else if (numpad5 || digit5) {
      onInput(currentChoiceIndex.value, 5)
    }
  }
)

async function onCalculate(data: { index: number; value: number }[]) {
  gaProxy.gtag('event', 'calculate', { scale: props.name })

  if (!checkScaleItemValidity({ minLimit: minLimit.value, maxLimit: maxLimit.value }, data)) return
  if (isLoading.value) return

  isLoading.value = true
  try {
    result.value = await $fetch('/api/scale', {
      method: 'POST',
      body: { scale: props.name, data },
    })

    setTimeout(() => goTo(choicesSlides.value.length), 300)
  } catch (error) {
    alert(error.message)
  }
  isLoading.value = false
}

function onPrint(_data: { index: number; value: number | null }[]) {}

function onClose() {
  gaProxy.gtag('event', 'model_test_close', { scale: props.name })
  result.value = undefined
  emit('close')
}

onBeforeUnmount(() => {})
</script>

<template>
  <ModalBase :is-open="isOpen" inner-class="w-[700px] p-4 overflow-hidden text-black dark:text-white" @close="onClose">
    <h4 class="mb-12 text-xl">{{ name }}</h4>

    <!-- Keyboard hint -->
    <div class="absolute right-16 top-4 flex w-fit flex-col gap-1 text-sm">
      <span>Use &#8592; / &#8594; keys to select {{ type === 'binary' ? 'True / False' : '1/2/3/4/5' }}</span>
      <span>Use &#8593; / &#8595; keys to move backward / forward</span>
      <span v-if="type === 'binary'">Use T/F keys to select True / False</span>
      <span v-else-if="type === 'pentanary'">Use 1/2/3/4/5 keys to select 1/2/3/4/5</span>
    </div>

    <!-- Keen Slider track -->
    <div ref="sliderRef" class="keen-slider">
      <!-- Question slides -->
      <div
        v-for="(groups, slideIndex) in choicesSlides"
        :key="`choice-${slideIndex}`"
        class="keen-slider__slide grid w-full grid-flow-col grid-rows-2 justify-items-center gap-6"
        :class="type === 'binary' ? 'grid-cols-3' : 'grid-cols-2'">
        <div v-for="(group, groupIndex) in groups" :key="groupIndex" class="flex flex-col gap-2">
          <template v-for="{ index, value } in group" :key="index">
            <InputChoice
              :options="options"
              :index="index"
              :value="value ?? undefined"
              :is-selected="currentChoiceIndex === index - 1"
              :is-invalid="invalidChoiceIndex === index - 1"
              @click="result = undefined"
              @update="(value) => onInput(index - 1, value as number)" />
          </template>
        </div>
      </div>

      <!-- Result slide — always mounted so Keen counts it; content shown when ready -->
      <div class="keen-slider__slide">
        <div v-if="result" class="relative max-h-[408px] overflow-y-scroll">
          <table class="w-[calc(100%-0.5rem)] table-auto overflow-hidden rounded-md">
            <tbody class="bg-light-400 dark:bg-dark-400">
              <tr v-for="({ name, value }, index) in result" :key="name">
                <td class="border-white p-2 pr-8 capitalize dark:border-black" :class="{ 'border-b': index !== result.length - 1 }">
                  {{ name.replaceAll('-', ' ') }}
                </td>
                <td class="border-x border-white p-2 pr-8 dark:border-black" :class="{ 'border-b': index !== result.length - 1 }">
                  {{ value }}
                </td>
                <td class="border-white p-2 pr-8 dark:border-black" :class="{ 'border-b': index !== result.length - 1 }" />
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- Controls: pagination + arrows/buttons -->
    <div class="mt-4 flex h-8 items-center justify-between">
      <!-- Pagination dots -->
      <ul class="absolute bottom-2 left-1/2 flex h-[10px] -translate-x-1/2 gap-1">
        <li v-for="(_, idx) in totalSlides" :key="idx" class="flex items-center justify-center">
          <button
            :class="['rounded-full transition-colors duration-300', idx === currentSlide ? 'h-[10px] w-[10px] bg-primary-500' : 'h-[6px] w-[6px] bg-white']"
            :aria-label="`Go to slide ${idx + 1}`"
            @click="goTo(idx)" />
        </li>
      </ul>

      <!-- Prev arrow — always visible -->
      <button :disabled="currentSlide === 0" class="scale-[-1] disabled:opacity-40" @click="goPrev">
        <NuxtIcon name="local:chevron-bold" />
      </button>

      <!-- Right side: Next arrow | Calculate | Print -->
      <template v-if="!isLastSlide">
        <!-- Hide the next button when on the result slide itself -->
        <button v-show="currentSlide < lastQuestionSlideIdx" class="disabled:opacity-40" @click="goNext">
          <NuxtIcon name="local:chevron-bold" />
        </button>
      </template>

      <AppButton
        v-if="isLastSlide && !result"
        :is-loading="isLoading"
        title="Calculate"
        size="M"
        class="ml-auto !px-3 !py-1 transition-[width] duration-300 ease-in-out"
        @click="onCalculate(choices)" />
      <AppButton v-if="isLastSlide && result" title="Print" size="M" class="ml-auto !px-3 !py-1 transition-[width] duration-300 ease-in-out" @click="onPrint(choices)" />
    </div>
  </ModalBase>
</template>

<style scoped>
/* Override Keen's default positioning so it doesn't fight ModalBase */
:deep(.keen-slider) {
  @apply overflow-hidden;
}

/* Arrows color passthrough */
:deep(.arrows) {
  @apply text-primary-400;
}
</style>
