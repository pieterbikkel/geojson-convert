<script setup lang="ts">
const { locale } = useI18n()

useHead({
  meta: [{ name: 'viewport', content: 'width=device-width, initial-scale=1' }],
  htmlAttrs: { lang: locale }
})
</script>

<template>
  <UApp>
    <NuxtPage />
  </UApp>
</template>
