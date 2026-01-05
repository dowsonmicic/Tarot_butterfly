<template>
  <div id="app-container">
    <!-- <TarotScene /> -->
    <TestCube />
  </div>
</template>

<script setup>
import TarotScene from './components/TarotScene.vue'
import TestCube from './components/TestCube.vue'
</script>

<style>
body {
  margin: 0;
  padding: 0;
  overflow: hidden;
  background-color: #050510;
}

#app {
  width: 100vw;
  height: 100vh;
}
</style>
