<script setup lang="ts">
import TravelAssistant from './components/TravelAssistant.vue'
</script>

<template>
  <TravelAssistant />
</template>

<style>
/* 全局样式重置 */
*,
*::before,
*::after {
  box-sizing: border-box;
}

html,
body {
  margin: 0;
  padding: 0;
  min-height: 100%;
}

body {
  font-family: system-ui, -apple-system, BlinkMacSystemFont, "SF Pro Text", "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
  line-height: 1.9;
  font-size: 20px;
}
</style>
