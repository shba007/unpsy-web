<script setup lang="ts">
const urls = ref([
  { url: '/dashboard', id: 'dashboard', title: 'Dashboard', icon: 'local:widget' },
  { url: '/dashboard', id: 'scanner', title: 'Scanner', icon: 'local:scanner' },
])
</script>

<template>
  <!-- TODO: Make it collapsable -->
  <nav
    class="relative bottom-0 z-50 flex w-screen shrink-0 flex-col overflow-hidden border-primary-500 bg-light-500 !fill-primary-500 px-[1.25rem] py-[0.75rem] !text-primary-500 dark:bg-dark-500 sm:w-[100px] sm:py-8 md:max-w-[252px] md:border-r lg:w-[252px]">
    <div class="mx-auto hidden w-fit items-center justify-center sm:flex">
      <NuxtIcon name="local:logo" class="text-[56px]" />
      <h1 class="font-sub text-[28px] uppercase">Unpsy</h1>
    </div>
    <ul class="md:max-h-1/2 z-10 flex grow gap-4 overflow-hidden sm:my-12 sm:w-[220px] sm:flex-col">
      <li v-for="{ url, id, title, icon } in urls" :key="id" :class="{ 'mt-auto self-center justify-self-end': id === 'logout' }">
        <NuxtLink
          :to="url"
          :replace="id !== 'dashboard'"
          class="flex cursor-pointer items-center gap-2 overflow-hidden rounded-2xl bg-light-400 p-2 font-semi-bold opacity-50 dark:bg-dark-600 sm:px-0"
          active-class="bg-dark-600 text-primary-400 transition-all duration-500 !opacity-100">
          <svg width="21" height="32" viewBox="0 0 21 32" fill="none" xmlns="http://www.w3.org/2000/svg" class="hidden sm:inline">
            <path d="M0 2C3.31371 2 6 4.68629 6 8V24C6 27.3137 3.31371 30 0 30V2Z" />
          </svg>
          <!-- <div class="" /> -->
          <NuxtIcon :name="icon" class="text-[32px]" />
          <span class="hidden whitespace-nowrap capitalize sm:inline">{{ title }}</span>
        </NuxtLink>
      </li>
    </ul>
  </nav>
</template>

<style scoped>
ul > li > a.router-link-exact-active > svg {
  @apply fill-primary-400;
}
</style>
