<script setup lang="ts">
const title = `Dashboard`
const description = `A psychoanalytical web tool for diverse assessments and automatic analysis with a built-in scanner. 
It includes various assessments scales like EPQ, MACI, MCMI, TCI, JEPQ.`
const {
  public: { siteUrl },
} = useRuntimeConfig()
const imageUrl = `${siteUrl}/previews/dashboard.webp`

definePageMeta({
  // middleware: 'auth',
  layout: 'app',
})

useSeoMeta({
  title: title,
  ogTitle: title,
  twitterTitle: title,
  description: description,
  ogDescription: description,
  twitterDescription: description,
  ogImage: imageUrl,
  twitterImage: imageUrl,
  ogUrl: `${siteUrl}/dashboard`,
})

useSchemaOrg([
  defineBreadcrumb({
    itemListElement: [{ name: 'Dashboard', item: '/dashboard' }],
  }),
])

// ── Data ─────────────────────────────────────────────────────────────────────
const { data: scales } = useFetch('/api/scale', { method: 'GET' })

// ── Responsive grid chunking ──────────────────────────────────────────────────
// Mirrors Splide grid: rows=2, cols=1/2/3 at 0/768/1280px
const { width } = useWindowSize()

const gridCols = computed(() => {
  if (width.value >= 1280) return 3
  if (width.value >= 768) return 2
  return 1
})

const ROWS = 2
const itemsPerSlide = computed(() => ROWS * gridCols.value)

const chunkedScales = computed(() => {
  const items = scales.value ?? []
  const size = itemsPerSlide.value
  const chunks: (typeof items)[] = []
  for (let i = 0; i < items.length; i += size) {
    chunks.push(items.slice(i, i + size))
  }
  return chunks
})

// ── Keen Slider ───────────────────────────────────────────────────────────────
const currentSlide = ref(0)
const totalSlides = computed(() => chunkedScales.value.length)

const [sliderRef, slider] = useKeenSlider({
  initial: 0,
  slideChanged(s) {
    currentSlide.value = s.track.details.rel
  },
})

// Re-initialise slider when the number of chunks changes (breakpoint resize)
watch(chunkedScales, async () => {
  await nextTick()
  currentSlide.value = 0
  slider.value?.update({}, 0)
})

const prev = () => slider.value?.prev()
const next = () => slider.value?.next()
const goTo = (idx: number) => slider.value?.moveToIdx(idx)

// ── Scale modal state ─────────────────────────────────────────────────────────
const selectedScaleName = ref<string | null>(null)
const selectedScale = computed(() => (selectedScaleName.value ? scales.value?.find(({ name }) => name === selectedScaleName.value) : undefined))
const openedModel = ref<'scale' | 'payment' | 'feedback' | null>(null)
</script>

<template>
  <section class="relative flex flex-col">
    <!-- Slider track -->
    <div ref="sliderRef" class="keen-slider py-2">
      <div v-for="(chunk, chunkIdx) in chunkedScales" :key="chunkIdx" class="keen-slider__slide">
        <!-- CSS grid replicates the Splide rows×cols layout -->
        <div class="grid gap-3" :style="{ gridTemplateColumns: `repeat(${gridCols}, minmax(0, 1fr))` }">
          <CardScale
            v-for="{ name, type, count, subScales, updatedAt, publishedAt } in chunk"
            :key="name"
            :name="name"
            :type="type"
            :count="count"
            :sub-scales="subScales"
            :updated-at="updatedAt"
            :published-at="publishedAt"
            @open-test="
              () => {
                selectedScaleName = name
                openedModel = 'scale'
              }
            "
            @open-payment="
              () => {
                selectedScaleName = name
                openedModel = 'payment'
              }
            " />
        </div>
      </div>
    </div>

    <!-- Controls: arrows + pagination -->
    <div class="mt-4 flex items-center justify-between">
      <!-- Pagination dots -->
      <ul class="flex gap-1">
        <li v-for="(_, idx) in chunkedScales" :key="idx" class="flex items-center">
          <button
            :class="['rounded-full transition-all duration-300', idx === currentSlide ? 'h-[10px] w-[10px] bg-primary-500' : 'h-[6px] w-[6px] bg-white']"
            :aria-label="`Go to slide ${idx + 1}`"
            @click="goTo(idx)" />
        </li>
      </ul>

      <!-- Prev / Next arrows -->
      <div class="ml-auto flex w-16 justify-between">
        <button class="disabled:opacity-40" :disabled="currentSlide === 0" aria-label="Previous" @click="prev">
          <NuxtIcon name="local:chevron-bold" class="rotate-180" />
        </button>
        <button class="disabled:opacity-40" :disabled="currentSlide === totalSlides - 1" aria-label="Next" @click="next">
          <NuxtIcon name="local:chevron-bold" />
        </button>
      </div>
    </div>

    <!-- Scale modal -->
    <ModalScale
      v-if="openedModel === 'scale' && selectedScale"
      :is-open="openedModel === 'scale'"
      :name="selectedScale.name"
      :type="selectedScale.type"
      :count="selectedScale.count"
      :options="selectedScale.options"
      @close="
        () => {
          selectedScaleName = null
          openedModel = null
        }
      " />
  </section>
</template>
