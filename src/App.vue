<script setup>
import PixelImage from './components/PixelImage.vue'
</script>

<template>
  <PixelImage />
</template>

<style>
* {
  box-sizing: border-box;
}

body,
html {
  margin: 0 !important;
  padding: 0 !important;
  width: 100%;
  height: 100%;
  overflow: hidden;
  background-color: #070a0c;
}

#app {
  width: 100%;
  height: 100%;
}
</style>
